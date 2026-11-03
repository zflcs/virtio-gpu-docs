# Intel GVT-g Mediated vGPU 源码跟踪

## 1. 目标与状态边界

对应方案比较：

- [GPU 虚拟化平台比较](../virtio-gpu/platform-comparison.md)
- [Intel Xe GPU 虚拟化与 SR-IOV](intel-xe-sriov.md)
- [VFIO、IOMMUFD、SR-IOV 与 mdev 源码](vfio-iommufd-sriov-source-trace.md)

GVT-g 是 i915 时代的软件 mediated vGPU：多个 Guest 运行原生 i915 驱动，
Host GVT-g 截获 MMIO/config/GTT/command，进行验证、shadowing 和时间片调度，
再通过 Host i915 使用物理 GPU。

固定源码：

```text
Linux 6b4b543fa18757297ac756a6fe6e64f506b7844a
drivers/gpu/drm/i915/intel_gvt.c
drivers/gpu/drm/i915/gvt/
```

源码状态必须严格区分：

- 固定源码中仍存在 GVT-g implementation；
- 支持设备检查只列出 Broadwell、Skylake、Kaby Lake、Broxton、Coffee Lake、
  Comet Lake；
- GuC submission 启用时 GVT-g 初始化直接拒绝；
- 这不证明当前发行版、现代 GPU 或产品仍正式支持 GVT-g；
- 没有本地实验证明任何具体 Windows/Linux Guest、driver 和 workload 可用。

本文把它作为成熟的软件 mediated GPU virtualization 案例，而不是推荐用于
当前 AI GPU 产品。

## 2. 一句话模型

```text
Guest i915 driver
  -> virtual PCI config/BAR/MMIO
  -> KVMGT mdev/VFIO + KVM page tracking
  -> GVT-g command shadow and GTT shadow
  -> vGPU workload scheduler
  -> Host i915 request
  -> physical GPU engine
```

与 VFIO 整卡的区别：

- Guest 不直接拥有物理 PCI function；
- Guest MMIO 不是透明 BAR mapping；
- Guest commands 被 Host parser 检查；
- Guest PPGTT 被 shadow 成 Host page tables；
- vGPU 直接共享物理 engine，由 GVT-g 时间片多路复用。

## 3. Host 启用条件

[`intel_gvt_init_device()`](../../linux/drivers/gpu/drm/i915/intel_gvt.c:154)
执行：

1. 检查 `enable_gvt` 参数；
2. Guest 模式中不再启用 Host GVT-g；
3. 只允许固定世代设备；
4. 拒绝 GuC submission；
5. 保存物理 GPU 初始 PCI config 和 MMIO 状态；
6. 调用 GVT device operations 初始化。

[`save_initial_hw_state()`](../../linux/drivers/gpu/drm/i915/intel_gvt.c:109)
保存：

- `PCI_CFG_SPACE_EXP_SIZE` config；
- 通过 `intel_gvt_iterate_mmio_table()` 枚举的 MMIO；
- 两块 1 MiB MMIO 存储。

这些 snapshot 是 Host 基线。vGPU context switch/reset 需要恢复受控 MMIO，
而不是把 Guest 值永久写进物理设备。

## 4. GVT-g 对象

| 对象 | 作用 |
| --- | --- |
| `struct intel_gvt` | Host 全局 GPU virtualization 状态 |
| `struct intel_vgpu` | 一个 Guest 的虚拟 GPU |
| `struct intel_vgpu_type` | vGPU profile 与 resource requirements |
| `struct intel_vgpu_mm` | Guest GGTT/PPGTT 描述和 shadow 状态 |
| `struct intel_vgpu_workload` | 一次可调度的 Guest submission |
| `struct intel_vgpu_submission` | per-vGPU ring/execlist/shadow engine state |
| KVMGT `vfio_device` | 交给 userspace/VFIO 的设备 object |

固定源码 `GVT_MAX_VGPU` 为 8，表示 table/resource limit；实际可创建数量还受
low GM、high GM、fences 和 type profile 约束。

## 5. vGPU profile 与资源

[`intel_vgpu_configs[]`](../../linux/drivers/gpu/drm/i915/gvt/vgpu.c:95)
静态定义四种 profile：

| 名称后缀 | low GM | high GM | fences | weight | EDID |
| --- | ---: | ---: | ---: | ---: | --- |
| 8 | 64 MiB | 384 MiB | 4 | 2 | 1024x768 |
| 4 | 128 MiB | 512 MiB | 4 | 4 | 1920x1200 |
| 2 | 256 MiB | 1024 MiB | 4 | 8 | 1920x1200 |
| 1 | 512 MiB | 2048 MiB | 4 | 16 | 1920x1200 |

这不是 SR-IOV VF quota。它们是 GVT-g 在共享 i915 driver 内模拟的 aperture、
hidden graphics memory、fence 和 scheduling weight。

[`intel_gvt_init_vgpu_types()`](../../linux/drivers/gpu/drm/i915/gvt/vgpu.c:109)
根据物理 GPU 可用 aperture/hidden GM 计算哪些 type 能出现。

### 5.1 资源不是“独立 VRAM”

Guest 看到的 GM 是虚拟资源，最终物理存储和 engine 仍由 Host GPU 管理。
两个 vGPU 即使 profile 相同，也可能共享：

- physical engine；
- L3/LLC；
- memory bandwidth；
- Host firmware/reset domain；
- display hardware。

## 6. vGPU 创建

[`intel_gvt_create_vgpu()`](../../linux/drivers/gpu/drm/i915/gvt/vgpu.c:314)
的顺序：

1. 分配 vGPU ID；
2. 初始化 scheduling weight 和 locks；
3. 初始化 virtual PCI config；
4. 初始化 MMIO；
5. 分配 low/high GM 和 fences；
6. 建立 PVINFO；
7. 初始化 GTT；
8. 初始化 opregion；
9. 初始化 display/EDID；
10. 初始化 submission；
11. 初始化 scheduling policy；
12. 建立 debugfs、opregion 和 EDID region；
13. 生成 MMIO register whitelist。

失败时按资源依赖反向释放。vGPU 创建不是一次 ioctl 完成，而是多个 Guest
可见状态和 Host shadow state 的一致建立。

## 7. KVMGT/mdev/VFIO 接入

[`intel_vgpu_probe()`](../../linux/drivers/gpu/drm/i915/gvt/kvmgt.c:1457)
为 mdev 分配 `vfio_device`，并调用：

```text
vfio_register_emulated_iommu_dev()
```

这说明 vGPU 是 emulated/mediated device，不是物理 PCI function。

### 7.1 Device ops

[`intel_vgpu_dev_ops`](../../linux/drivers/gpu/drm/i915/gvt/kvmgt.c:1440)
实现：

- init/release；
- open/close；
- read/write；
- mmap；
- VFIO ioctl；
- region info；
- DMA unmap；
- IOMMUFD emulated bind/attach/detach。

### 7.2 KVMGT page tracking

[`intel_vgpu_open_device()`](../../linux/drivers/gpu/drm/i915/gvt/kvmgt.c:663)：

- 同一 KVM 中拒绝不期望的重复 vGPU attach；
- 注册 KVM page-track notifier；
- 激活 vGPU。

Guest 写被跟踪的 page table 页时，KVMGT 得到 write notification，GVT-g 再更新
shadow PPGTT。该机制要求 Guest 必须关联 KVM；不是任意 userspace VFIO
consumer 都能获得同等语义。

### 7.3 Guest page pin 与 DMA cache

KVMGT 维护 GFN -> HVA -> DMA address cache：

- `gvt_pin_guest_page()`；
- `gvt_dma_map_page()`；
- `gvt_dma_unmap_page()`；
- gfn cache 和 dma-address cache；
- VFIO DMA unmap callback。

这使 Host GVT-g 能把 Guest graphics memory 描述转换成 Host DMA address。

## 8. PCI config、BAR 和 MMIO

[`intel_vgpu_read()`](../../linux/drivers/gpu/drm/i915/gvt/kvmgt.c:866)和
[`intel_vgpu_write()`](../../linux/drivers/gpu/drm/i915/gvt/kvmgt.c:940)
按 VFIO region：

- config -> `intel_vgpu_emulate_cfg_*()`；
- BAR0 -> virtual MMIO；
- BAR2 -> aperture；
- 其他 BAR/VGA/ROM 受控返回；
- vendor region -> EDID/opregion 等。

[`intel_vgpu_ioctl()`](../../linux/drivers/gpu/drm/i915/gvt/kvmgt.c:1259)
提供 VFIO device info、IRQ、reset 和 graphics plane/dma-buf ioctl。

### 8.1 BAR2 aperture

BAR2 通过 sparse mmap 暴露 vGPU aperture range。
[`intel_vgpu_aperture_rw()`](../../linux/drivers/gpu/drm/i915/gvt/kvmgt.c:773)
仍检查 offset 是否在 vGPU aperture 内，并把访问映射到 Host GGTT iomap。

所以即使 Guest 能 mmap aperture，它也不是任意 Host GGTT 地址。

### 8.2 MMIO

[`intel_vgpu_emulate_mmio_read()`](../../linux/drivers/gpu/drm/i915/gvt/mmio.c:113)
和
[`intel_vgpu_emulate_mmio_write()`](../../linux/drivers/gpu/drm/i915/gvt/mmio.c:188)
根据 MMIO descriptor 分派到 read/write handlers。

未知/特权 MMIO 不能直接落到物理硬件。注册表 handler 是安全边界的一部分。

## 9. GTT 与 shadow PPGTT

### 9.1 Guest 页表不能直接成为 Host 页表

Guest GPU PTE 引用 Guest physical/GMA；物理 i915 需要 Host DMA address。
GVT-g 因此构建 shadow page table。

[`shadow_ppgtt_mm()`](../../linux/drivers/gpu/drm/i915/gvt/gtt.c:1723)：

1. 遍历 Guest PDP；
2. 为 Guest page-table page 分配 shadow SPT；
3. 生成 Host shadow entry；
4. 建立 shadow root entry。

### 9.2 Guest PTE write

[`ppgtt_handle_guest_write_page_table()`](../../linux/drivers/gpu/drm/i915/gvt/gtt.c:1510)
先加新 entry，再移除旧 entry，确保切换窗口内页表仍有效。

write 可能被拆成：

- 完整 entry write；
- partial write/post-shadow；
- out-of-sync page；
- scratch PTE。

### 9.3 Out-of-sync 与 post-shadow

高频 Guest PTE updates 可以使用 out-of-sync page：

- guest PTE 先写入 OOS page；
- 暂停对该 GFN 的 page track；
- workload 提交前同步全部 OOS page；
- partial write 进入 post-shadow bitmap，在 workload prepare 时 flush。

这是用 batching 减少每次 Guest PTE write 的 Host shadow 成本，但增加了
 synchronization/lifetime 复杂度。

## 10. Guest command 扫描

### 10.1 复制 guest ring

[`shadow_workload_ring_buffer()`](../../linux/drivers/gpu/drm/i915/gvt/cmd_parser.c:2928)
处理 ring wrap：

```text
head -> top
start -> tail
```

把 Guest GMA ring 复制到 kernel scan buffer。

### 10.2 opcode decode

[`cmd_parser_exec()`](../../linux/drivers/gpu/drm/i915/gvt/cmd_parser.c:2734)：

- hash lookup opcode；
- unknown command 返回 `-EBADRQC`；
- variable-fixed length 必须匹配合法值；
- 调用 register/address/display-flip/interrupt handlers；
- 推进 instruction pointer。

### 10.3 address 与 privilege 检查

Command hander 会检查：

- GMA/GPU address 是否位于合法 ring/buffer；
- batch buffer start/end；
- privileged register write；
- address range overflow；
- display flip 的 surface/plane 条件。

Host parser 不是“性能日志工具”，而是 Guest 到物理 GPU 的安全过滤器。

### 10.4 Batch buffer shadow

Privilege batch buffer 可能被复制到 shadow BO，并替换 ring 中地址；非 privilege
batch buffer 可以走原 GMA path，但仍经过扫描验证。

scan error 可把 vGPU 标记 unhealthy/failsafe，停止继续向硬件提交。

## 11. Execlist 和 workload

### 11.1 Guest ELSP

[`intel_vgpu_submit_execlist()`](../../linux/drivers/gpu/drm/i915/gvt/execlist.c:456)
解析两个 ELSP descriptor：

- descriptor 0 必须 valid；
- privilege access 必须正确；
- 为 valid context 创建 workload；
- 模拟 Guest scheduler status/context switch；
- 把 workload 加入 vGPU engine queue。

### 11.2 Workload 创建

[`intel_vgpu_create_workload()`](../../linux/drivers/gpu/drm/i915/gvt/scheduler.c:1635)
从 Guest LRCA/ring context 读取：

- ring head/tail/start/control；
- indirect context；
- per-context workaround context；
- Guest PPGTT；
- engine/context descriptor。

验证 GMA 范围后建立 workload。

### 11.3 Shadow 和 prepare

[`intel_gvt_scan_and_shadow_workload()`](../../linux/drivers/gpu/drm/i915/gvt/scheduler.c:494)：

1. 更新 shadow context descriptor；
2. copy + scan ring；
3. scan/shadow workaround indirect context；
4. 标记 shadow 完成。

[`prepare_workload()`](../../linux/drivers/gpu/drm/i915/gvt/scheduler.c:746)：

- pin shadow MM；
- 更新 shadow PDP；
- 设置 shadow context PPGTT；
- sync out-of-sync pages；
- flush post-shadow；
- copy workload 到 Host ring；
- prepare batch/WA context。

### 11.4 Dispatch

[`dispatch_workload()`](../../linux/drivers/gpu/drm/i915/gvt/scheduler.c:808)：

- 创建 Host i915 request；
- scan/shadow；
- populate shadow context；
- prepare；
- `i915_request_add()` 到 Host scheduler；
- 等待 completion 后回写 Guest context/status。

Guest 看到 virtual engine 完成，但实际 Host request 由 GVT-g 代提交。

## 12. 时间片调度

[`sched_policy.c`](../../linux/drivers/gpu/drm/i915/gvt/sched_policy.c)
实现 time-based scheduler：

- default timer period 为 1 ms；
- 每 100 ms 按 vGPU weight 重新分配 timeslice；
- 未使用/超用 timeslice 在下一阶段保留 debt/credit；
- 新 vGPU 启动有 2 秒优先调度；
- 只选择有 pending workload 的 active vGPU；
- 当前 Host workload 完成后才能切换 vGPU。

[`intel_gvt_schedule()`](../../linux/drivers/gpu/drm/i915/gvt/sched_policy.c:241)
计算 consumed timeslice、选择 next vGPU，再更新 current vGPU。

这不是 SR-IOV hardware VF scheduler，也不是 MIG partition；所有 vGPU 最终共享
同一 Host engine，由软件 policy 多路复用。

## 13. MMIO context switch

[`intel_gvt_switch_mmio()`](../../linux/drivers/gpu/drm/i915/gvt/mmio_context.c:571)
在 vGPU 切换时：

1. 保存前一个 vGPU 的 engine MMIO；
2. 更新 MOCS/L3/LLC policy state；
3. 加载下一个 vGPU 的 engine MMIO；
4. 必要时处理 TLB pending；
5. 维护 engine owner。

如果 MMIO 切换不完整，vGPU B 可能看到 vGPU A 的 render state、MOCS 或 engine
context，导致错误数据甚至跨租户影响。

## 14. 中断、显示与 dma-buf

### 14.1 Interrupt

GVT-g virtual interrupt state 将 Host GPU event 投影到 vGPU。guest interrupt
不能被直接视为 Host MSI-X 向量；需要 event mask/status 和 scheduling 边界。

### 14.2 Display

[`intel_vgpu_query_plane()`](../../linux/drivers/gpu/drm/i915/gvt/dmabuf.c:404)
解析 Guest framebuffer/plane/cursor metadata。
[`intel_vgpu_get_dmabuf()`](../../linux/drivers/gpu/drm/i915/gvt/dmabuf.c:501)
创建 Host GEM 并导出 dma-buf 给 VMM/display backend。

[`vgpu_create_gem()`](../../linux/drivers/gpu/drm/i915/gvt/dmabuf.c:199)
建立 guest backing resource 的 Host GEM view。display path 还需验证：

- plane format/stride；
- cursor hotspot；
- framebuffer size；
- dma-buf lifetime；
- Guest unbind 与 Host display 使用之间的 fence。

## 15. Reset 与生命周期

[`intel_gvt_reset_vgpu_locked()`](../../linux/drivers/gpu/drm/i915/gvt/vgpu.c:437)
区分：

- DMLR：Device Model Level Reset，模拟 PCI-level full reset；
- GT reset：full 或 per-engine reset。

流程包括：

1. 停止 vGPU schedule；
2. 等待当前 workload idle；
3. reset submission；
4. 选择 submission ops；
5. invalidate PPGTT；
6. reset virtual config/MMIO/display state；
7. 重建后续可执行状态。

`intel_gvt_destroy_vgpu()` 反向清理 scheduler、submission、display、opregion、
GGTT/GTT、MMIO、dmabuf。

固定源码中没有通用 QEMU migration state machine。不能从 GVT-g reset 推出
vGPU live migration 可用。

## 16. 安全边界

### 16.1 保护

- Guest MMIO/config 需 handler；
- command parser 拒绝 unknown/invalid/privileged command；
- PPGTT shadow 把 Guest physical 转成 Host DMA；
- KVM page tracking 拦截 Guest page table write；
- vGPU scheduler 隔离 software context 时序。

### 16.2 仍需共享/验证

- Host i915/firmware；
- physical engines；
- L3/LLC/memory bandwidth；
- display；
- reset 域；
- parser 的历史漏洞；
- Host kernel 侧 vGPU object lifetime。

### 16.3 不能推出的结论

- mdev 存在就代表硬件隔离；
- parser 检查就代表不存在侧信道；
- host dma-buf 可导出就代表跨租户安全；
- GVT-g reset 就代表其他 vGPU 不受影响。

## 17. 与 virtio-gpu、Xe SR-IOV、VFIO 对照

| 维度 | GVT-g | virtio-gpu | Xe SR-IOV |
| --- | --- | --- | --- |
| Guest device | Intel virtual PCI vGPU | `1af4:1050` | PCI VF |
| Guest driver | Native i915 | virtio-gpu + capset UMD | Native Xe VF |
| Host interception | MMIO/config/GTT/command | virtio command/renderer | PF/GuC/firmware + VF resource |
| GPU command | Host parser/scan | Host renderer/proxy parser | Guest VF driver commands |
| Memory | shadow GTT/PPGTT + KVMGT pin | resource/blob/ring | VF GPU VM + Host IOMMU |
| Scheduling | GVT-g software timeslice | Host GPU per renderer | PF/GuC/firmware VF scheduling |
| Isolation type | software mediated | software renderer boundary | PCI/function + firmware |
| mdev/VFIO | KVMGT mdev | QEMU virtual device | VFIO VF |

GVT-g 是最接近“Guest 原生 i915 + Host mediation”的旧方案，但它是整条复杂
command/page-table path 的软件 shadow，不是现代 hardware VF。

## 18. 对 AI compute 的意义

GVT-g 主要虚拟化 i915 graphics command 和 display/context model，不能据此
推出：

- 支持完整 Level Zero/oneAPI compute；
- 支持 Intel GPU AI framework 的全部 kernel/library；
- 支持 SVM/page fault；
- 支持 MIG/vGPU profile；
- 支持跨 GPU collective。

对 AI 学习者的价值是架构对照：

- 为什么 command parser/shadow 有高 CPU 成本；
- 为什么 native Guest driver 需要 Host GTT/MMIO virtualization；
- 为什么 time slicing 不等于 VRAM/engine partition；
- 为什么现代 Intel 转向 Xe SR-IOV/PF-VF provisioning；
- 为什么 virtio-gpu API remoting 与 GVT-g 是两种不同 trust boundary。

现代 AI compute 应优先研究 Xe SR-IOV、VFIO 原生栈、Level Zero/oneAPI 全栈兼容
和固定产品支持矩阵，而不是把 GVT-g 当作当前 AI GPU 方案。

## 19. 失败矩阵

| 症状 | 优先检查 |
| --- | --- |
| GVT-g module 未初始化 | enable_gvt、设备世代、GuC submission |
| mdev 类型不存在 | vGPU type 初始化、GM/fence 资源 |
| vGPU create 失败 | GTT/opregion/display/submission resource rollback |
| Guest 驱动探测失败 | virtual PCI config/BAR/opregion/EDID |
| MMIO write 被拒绝 | register handler、whitelist、privilege |
| Guest page table 不更新 | KVM page tracking、post-shadow/OOS sync |
| GPU 不执行 command | execlist descriptor、scan error、Host i915 request |
| 某个 Guest 卡住 | workload queue、current vGPU、Host engine hang |
| 跨 vGPU 数据异常 | MMIO context/MOCS restore、PPGTT/GTT、scratch PTE |
| display 黑屏 | framebuffer decode、dma-buf export、plane/EDID |
| reset 后驱动异常 | DMLR/GT reset、config/MMIO restore、PPGTT invalidate |

## 20. 源码阅读顺序

1. `intel_gvt.c`：Host enable 和初始 HW state。
2. `gvt/gvt.h`：GVT/vGPU/GM/fence/MMIO 对象。
3. `vgpu.c`：type、resource、create/destroy/reset。
4. `kvmgt.c`：mdev/VFIO、KVM tracking、DMA cache、BAR/region。
5. `mmio.c`、`cfg_space.c`：Guest register emulation。
6. `gtt.c`：GGTT/PPGTT shadow、OOS、post-shadow。
7. `cmd_parser.c`：command decode、address/register audit、ring shadow。
8. `execlist.c`、`scheduler.c`：workload 创建、prepare、dispatch、completion。
9. `sched_policy.c`：weight/time-slice vGPU 调度。
10. `mmio_context.c`：vGPU engine/MOCS state switch。
11. `dmabuf.c`、`display.c`：scanout 和 dma-buf。

## 21. 检查题

1. GVT-g 为什么既不是 VFIO 整卡，也不是 virtio-gpu？
2. mdev framework 与 GVT-g shadow engine 分别负责什么？
3. Host 为什么保存 initial config 和 MMIO snapshot？
4. vGPU profile 的 low/high GM 是什么资源，不是物理 VRAM 的什么？
5. KVM page tracking 为什么对 Guest PPGTT 必需？
6. out-of-sync page 和 post-shadow 分别解决什么路径成本？
7. command parser 如何阻止 Guest 提交通往其他地址的命令？
8. 为什么 Guest ELSP descriptor 必须经过 privilege 检查？
9. workload shadow/prepare/dispatch 的锁和 fence 顺序是什么？
10. GVT-g 的 100 ms timeslice balance 与 Xe VF 调度差在哪？
11. context switch 时漏掉 MOCS/MMIO restore 会造成什么？
12. DMLR 与 GT reset 的 scope 有何区别？
13. GVT-g reset 为什么不等于 live migration？
14. 为什么 parser 存在仍不能宣称完整租户安全？
15. GVT-g 对现代 Level Zero/SYCL AI workload 的证据缺口是什么？

## 22. 与其他专题

- [Intel Xe GPU 虚拟化与 SR-IOV](intel-xe-sriov.md)
- [Intel Xe SVM、Userptr 与 GPU Page Fault 源码跟踪](intel-xe-svm-source-trace.md)
- [VFIO、IOMMUFD、SR-IOV 与 mdev 源码](vfio-iommufd-sriov-source-trace.md)
- [QEMU VFIO 源码跟踪](qemu-vfio-source-trace.md)
- [GPU 虚拟化平台比较](../virtio-gpu/platform-comparison.md)
- [OpenCL、Level Zero、SYCL 与 AI compute 虚拟化边界](opencl-levelzero-sycl-virtualization.md)
