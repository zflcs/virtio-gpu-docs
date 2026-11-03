# Intel Xe GPU 虚拟化与 SR-IOV

## 1. 文档定位

本文补 Intel Xe 这一厂商路径，重点分析 Linux Xe KMD 公开实现的：

- tile/GT/engine 模型；
- execution queue、VM、BO；
- SR-IOV PF/VF；
- VF provisioning 和 GuC 配置；
- time-slice 调度、VRAM quota 和 VF stop/FLR；
- VF migration；
- VFIO 与 sysfs/debugfs/configfs 管理面。

它不把 Intel 的语义外推到 NVIDIA MIG/vGPU 或 AMD SR-IOV；这些机制都在
[平台比较](../virtio-gpu/platform-comparison.md) 中分别讨论。

源码基线：Linux `6b4b543fa18757297ac756a6fe6e64f506b7844a`。

本文不运行 Xe GPU、VF、VM、migration 或性能实验。

## 2. Xe 的硬件抽象

### 2.1 Tile

[`xe_tile.c`](../../linux/drivers/gpu/drm/xe/xe_tile.c:27) 把 Intel tile 解释为
“类似一张独立 GPU”的硬件复制单元。多 tile 设备可能拥有：

- per-tile VRAM；
- per-tile GGTT；
- per-tile interrupt/migration；
- 一个或多个 GT；
- 通过 root tile 汇总 interrupt 和 memory region。

这与“PCI function = device”不完全同层：一个 PCI Xe device 可以包含多个 tile。

### 2.2 GT

GT 是执行/图形引擎的功能域：

- primary GT：render/compute 等；
- media GT：media 功能。

configfs 还允许 probe 前限制 `gt_types_allowed` 和 `engines_allowed`，用于
平台启用/debug；不是普通租户管理接口。

### 2.3 GuC

Xe 使用 GuC 作为重要 firmware submission/control 组件：

- execution queue submission；
- context/doorbell ID；
- PF/VF 配置 KLV；
- PF2GUC/VF2PF 管理请求；
- migration save/restore；
- engine activity accounting。

SR-IOV 的很多资源不是只写在 Host driver 内存里，还由 PF 通过 GuC 配置到
firmware。

## 3. Execution Queue

[`xe_exec_queue.c`](../../linux/drivers/gpu/drm/xe/xe_exec_queue.c:23) 定义：

- execution queue 是 HW context 的提交接口；
- queue 关联指定 engine 或同 tile/engine class 的 engine group；
- queue 属于一个 VM/address space；
- queue 持有 VM 和 LRC（Logical Ring Context）引用；
- submission backend 可透明使用 GuC 或 Execlist。

用户提交路径：

```text
DRM_IOCTL_XE_EXEC_QUEUE_CREATE
  -> bind to VM
  -> set engine/priority/timeslice
  -> DRM_IOCTL_XE_EXEC
  -> GuC/Execlist backend
  -> GPU engine
```

### 3.1 Multi Queue Group

多队列组只支持 GuC backend，用于 CCS/BCS：

- 一个 group 内 queues 共享同一 VM；
- primary queue 的属性控制整个 group；
- secondary queue 可动态加入；
- group 最多 64 queues；
- queue 被用户销毁后，KMD 仍保留 LRC 引用，直到 group 销毁；
- group 内一 queue 出错会触发整个 group context cleanup。

这说明 queue 资源的生命周期可能比单个用户 queue handle 更长，迁移和 reset
不能只按 queue 列表释放。

## 4. VM 与 BO

### 4.1 VM_BIND

Xe UAPI 使用：

- `DRM_IOCTL_XE_VM_CREATE`；
- `DRM_IOCTL_XE_VM_BIND`；
- BO map/unmap/prefetch；
- fault mode/long-running VM；
- userptr bind。

常见 bind flags：

- READONLY；
- IMMEDIATE；
- NULL；
- CPU_ADDR_MIRROR；
- MADVISE_AUTORESET；
- DECOMPRESS。

### 4.2 BO placement

[`xe_bo_doc.h`](../../linux/drivers/gpu/drm/xe/xe_bo_doc.h:8) 说明：

- TTM 管理 placement/eviction；
- kernel BO 通常 pinned/contiguous，映射在 GGTT；
- user BO 可 evict；
- private BO 绑定同一 VM，不能 PRIME export；
- external BO 可 dma-buf/PRIME 共享到多个 VM/device；
- eviction 通过 TTM 和 per-GT migration engine；
- suspend 时 VRAM 可能掉电，因此非 pinned user BO 移到 sysmem。

### 4.3 VM/BO 与 VF

VF Guest/Host driver 仍要遵循 Xe VM/BO 生命周期。SR-IOV 切分的是 PF 向
VF 暴露的 GGTT/context/doorbell/LMEM 等资源，不等于把每个 BO 都做成独立
物理设备。

## 5. PF/VF 检测和启用

### 5.1 PF/VF probe

[`xe_sriov_probe_early()`](../../linux/drivers/gpu/drm/xe/xe_sriov.c:48)：

- PF 通过 PCI `dev_is_pf()` 检测；
- VF 通过专用 MMIO register 检测；
- 平台 `has_sriov` 与 PF readiness 共同决定模式；
- 若平台未启用 SR-IOV 但 PCI capability 报告 totalvfs，driver 会把 totalvfs
  设为 0，避免 PCI layer 暴露不受支持的 VF。

这说明“PCI 有 SR-IOV cap”不证明当前 Xe driver/platform 支持该 VF。

### 5.2 VF 启用

[`xe_pci_sriov.c`](../../linux/drivers/gpu/drm/xe/xe_pci_sriov.c)：

- PF 保持 runtime PM active，因为 VF 不实现 PM capability；
- provision VF resources；
- 离散 GPU 调整 VF LMEM BAR size；
- `pci_enable_sriov()`；
- PF/VF device link 保证 resume 顺序；
- 启用 engine activity statistics；
- 禁用时关闭 statistics、unlink sysfs、disable SR-IOV、重新 provisioning、
  reset VFs 并释放资源。

## 6. VF Provisioning

### 6.1 GuC 配置 KLV

[`xe_gt_sriov_pf_config.c`](../../linux/drivers/gpu/drm/xe/xe_gt_sriov_pf_config.c)
显示 PF 通过 KLV 向 GuC 下发：

- GGTT start/size；
- context ID begin/count；
- doorbell ID begin/count；
- execution quantum；
- preemption timeout；
- scheduling priority；
- LMEM size；
- resource thresholds。

这些配置是 VF 能提交执行的资源边界。

### 6.2 sysfs profile

[Xe SR-IOV sysfs ABI](../../linux/Documentation/ABI/testing/sysfs-driver-intel-xe-sriov)
公开：

- `exec_quantum_ms`；
- `preempt_timeout_us`；
- `sched_priority`；
- `vram_quota`；
- `stop`。

调度参数适用于 time-sliced GPU sharing，可反映所有 tiles/GTs 的配置。

### 6.3 VRAM quota

`vram_quota` 仅在带 VRAM 的离散平台可见：

- 可在 VF 启用前作初始 provisioning；
- 非零初始值会阻止自动 provisioning；
- VFs 启用后可按硬件/FW 支持动态修改；
- 禁用 VFs 后 provisioning 释放；
- 实际值可能按硬件/FW 粒度对齐。

因此 sysfs 写入值不一定逐字节等于最终可用 VRAM。

### 6.4 调度优先级

文档定义：

- `low`：只有提交了 work 的函数在 quantum 中获调度；
- `normal`：无论是否提交 work 都按 quantum 调度；
- `high`：有 work 时进入下一 quantum 优先。

这是 GPU time-slice 共享参数，不等于 PCIe 隔离或 MIG partition。

## 7. VF Stop、FLR 和状态机

### 7.1 stop

VF sysfs `stop`：

- 立即停止该 function 的所有 GT execution；
- 再次调度需要 VF FLR。

这可用于处理 misbehaving VF，但意味着 stop 不是无损 pause。

### 7.2 pause/resume

[`xe_gt_sriov_pf_control.c`](../../linux/drivers/gpu/drm/xe/xe_gt_sriov_pf_control.c)
公开 VF 状态机：

```text
READY/RUNNING
  -> PAUSE_WIP -> PAUSED
  -> RESUME_WIP -> READY

any abnormal/host action
  -> STOP_WIP -> STOPPED

FLR phases:
  PREPARE -> START -> WAIT_GUC -> GUC_DONE
  -> SYNC -> RESET_CONFIG/DATA/MMIO -> FINISH
```

FLR 不只是置一个 `reset` flag；它要同步 firmware、配置、数据和 MMIO。

### 7.3 reset 资源

需要处理：

- VF queue/context；
- doorbell/context ID；
- GGTT；
- LMEM/VRAM；
- GuC state；
- interrupt/event；
- PF 记录的 VF state machine。

## 8. VF Migration

### 8.1 前提

[`xe_sriov_pf_migration.c`](../../linux/drivers/gpu/drm/xe/xe_sriov_pf_migration.c)
在 migration init 时检查：

- platform 有 memory-based IRQ；
- 该 device 未 disable migration；
- GuC migration support 版本满足要求；
- per-GT/每 VF migration ring 和 packet state 可初始化。

这是“代码路径存在”与“当前设备支持”之间的 hard gate。

### 8.2 保存内容

per-GT migration save 包括：

- GuC state；
- GGTT data；
- MMIO state；
- VRAM/LMEM chunks。

PF 用 migration ring 把 packet 交给 userspace save consumer；restore 时
userspace 作为 producer 写回 packet。

### 8.3 VF restore recovery

[`xe_gt_sriov_vf.c`](../../linux/drivers/gpu/drm/xe/xe_gt_sriov_vf.c) 显示 VF
restore 后还要：

- 重新查询 self config；
- 修正 GGTT address；
- rebase GuC context hardware state pointers；
- 执行 post-migration fixups；
- 等待 PF/GuC 完成状态；
- 在 fixups 完成后重新允许执行。

这说明 migration 数据复制只是过程的一部分，restore 后的资源 fixup 才是
设备重新可用的关键。

### 8.4 VFIO glue

[`xe_sriov_vfio.c`](../../linux/drivers/gpu/drm/xe/xe_sriov_vfio.c) 向
`xe-vfio-pci` 模块导出：

- migration supported；
- FLR wait/prepare；
- suspend/resume；
- stop-copy enter/exit；
- resume-data enter/exit；
- migration size；
- userspace migration data read/write；
- error stop。

这说明 Xe SR-IOV 与 VFIO migration UAPI 是组合关系，而不是同一个机制。

## 9. Configfs

[`xe_configfs.c`](../../linux/drivers/gpu/drm/xe/xe_configfs.c:10) 提供设备
probe 前配置：

- survivability mode；
- allowed GT types；
- allowed engines；
- PSMI；
- VRAM page offline logging；
- context restore BB；
- migration ULLS period。

它是 Host 平台启用/debug 面，不应暴露给不可信 Guest。限制 GT/engine 是
debug/平台配置，不是 SR-IOV VF 的通用调度 policy。

## 10. Sysfs/Debugfs

### 10.1 sysfs

`sriov_admin/` 只在：

- device 支持 SR-IOV；
- 是 PF；
- Xe PF driver support 开启时出现。

其中：

- `pf/`、`vf<n>/`；
- profile 参数；
- VF `stop`；
- device symlink；
- bulk profile。

### 10.2 debugfs

PF debugfs 暴露：

- VFs summary/version；
- pause/resume/stop/reset/save/restore；
- migration data/size；
- per-tile/GT state。

这些是高权限运维接口，不是普通租户控制面。

## 11. VFIO/VM 架构

### 11.1 VM + VFIO VF

```text
Host Xe PF
  -> VF
  -> vfio-pci / xe-vfio-pci glue
  -> VM
  -> Guest Xe driver
  -> Guest exec queues/VM binds/BOs
```

Guest 使用 Xe native UAPI，Host PF/GuC 管理 VF resource 和 firmware。

### 11.2 IOMMU

VF 的 system-memory DMA 仍需 Host IOMMU 限制。SR-IOV 资源 provisioning 不能
替代 IOMMU；反之，IOMMU 也不能防止 Guest 用错误 Xe command 破坏共享 firmware
状态。

### 11.3 MIG?

MIG 是 NVIDIA 的内部 partition 术语。Intel VF 是 PCIe function + Xe/GuC
provisioning，不应称为 MIG。

## 12. 安全边界

需要检查：

- Guest 只能看到分配给自己的 VF；
- GGTT/context/doorbell/LMEM 范围不重叠；
- VF stop/FLR 真正停止旧 queue/DMA；
- VF BAR/resize 与 profile 一致；
- migration data 只属于对应 VF；
- PF sysfs/debugfs 不暴露普通租户；
- configfs 最终以 Host 权限使用；
- IOMMU/ACS/P2P 与 VF 组合验证；
- reset 不破坏 sibling VF（除非产品契约声明）。

## 13. 失败排查

| 症状 | 优先检查 |
| --- | --- |
| VF 不出现 | platform has_sriov、PF readiness、totalvfs、PCI enable |
| VF BAR 太小 | LMEM BAR resize、provision |
| VF 没有 VRAM | quota、auto provisioning、硬件粒度 |
| VF 不能执行 | GGTT/context/doorbell KLV、GuC ID range |
| VF 性能不公平 | exec quantum/preempt/priority |
| VF stop 后不恢复 | 必须 FLR |
| migration unavailable | memirq、GuC version、support gate |
| restore 后 VF hang | GGTT rebase/context HWSP fixup/self config |
| VF reset 影响其他 VF | PF/GuC/reset 域、firmware bug |

## 14. 性能关注

- VF time-slice quantum；
- preemption timeout；
- scheduling priority；
- LMEM/VRAM quota；
- shared memory bandwidth；
- shared GuC/command processing；
- VF BAR/GGTT coverage；
- migration chunk size/ULLS；
- multi-tile locality；
- queue class/engine sharing。

“有 VF”不等于性能是整卡的固定分数；time-slice、memory bandwidth、engine/
firmware 共享都影响结果。

## 15. 与 AMD/NVIDIA 对照

| 维度 | Intel Xe SR-IOV | AMD SR-IOV | NVIDIA MIG/vGPU |
| --- | --- | --- | --- |
| PCI VF | 是 | 是（支持平台） | vGPU/SR-IOV 依产品 |
| GPU 内部分区 | Xe/GuC LMEM/context/doorbell/VRAM policy | PF/AQL/HQD/VRAM | MIG hardware instance |
| Host firmware | GuC KLV/migration | MEC/MES/PSP etc. | GSP/vGPU manager |
| Guest driver | Xe VF driver | amdgpu/KFD VF | NVIDIA vGPU/MIG client |
| 主要 source | Xe KMD public | amdgpu/KFD public | product docs, partial kernel |

## 16. 检查题

1. Xe tile、GT、engine 和 VF 分别是什么层？
2. Execution queue 为什么绑定 VM，并持有 LRC/VM 引用？
3. Multi Queue Group 为什么延长 LRC 生命周期？
4. VF provisioning 为什么由 GuC KLV 而不只由 Host 内存表表达？
5. `exec_quantum_ms`、`preempt_timeout_us` 和 `sched_priority` 各控制什么？
6. `vram_quota` 为什么可能被硬件/FW 对齐？
7. VF `stop` 为什么必须 FLR 才能恢复？
8. Xe VF migration 为什么需要 memirq 和特定 GuC 版本？
9. GGTT/MMIO/VRAM save 之外，restore 后还需要哪些 fixup？
10. Xe configfs 为什么不是 Guest 租户接口？
11. SR-IOV VF 与 IOMMU、MIG、vGPU 的边界差别是什么？

## 17. 与其他专题

- 平台机制比较：  
  [GPU 虚拟化平台比较](../virtio-gpu/platform-comparison.md)
- queue/VM/fence：  
  [GPU 执行模型、队列、调度与故障隔离](gpu-execution-model.md)
- BO/VRAM/迁移：  
  [GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)
- SVM/userptr/fault：  
  [Intel Xe SVM 源码跟踪](intel-xe-svm-source-trace.md)
- 旧式软件 mediated vGPU：  
  [Intel GVT-g 源码跟踪](intel-gvt-g-source-trace.md)
- reset/migration：  
  [GPU VM 生命周期、热插拔、Checkpoint 与迁移](vm-lifecycle-migration.md)
- 安全：  
  [GPU 虚拟化的隔离、安全与故障域](gpu-virtualization-security.md)
