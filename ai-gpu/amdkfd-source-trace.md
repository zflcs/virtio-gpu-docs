# AMD KFD/ROCr 计算路径源码跟踪

## 1. 目的与范围

对应总览：[AI GPU 计算栈与虚拟化边界](ai-compute-virtualization.md)。

本文跟踪 AMD ROCm compute 从 `/dev/kfd` 到 GPU queue、doorbell、event、
memory 和 SVM 的 kernel 路径。它补齐“图形 API remoting 之后，AI compute
真正需要虚拟化什么”这一层。

源码基线：

- Linux `6b4b543fa18757297ac756a6fe6e64f506b7844a`
- `include/uapi/linux/kfd_ioctl.h`
- `drivers/gpu/drm/amd/amdkfd/`

本文不运行 ROCm 或 GPU workload。

## 2. 从 AI framework 到 KFD

概念路径：

```text
PyTorch/JAX/etc.
  -> HIP runtime / ROCr
  -> libhsakmt
  -> /dev/kfd
  -> KFD process representative
  -> compute/AQL/SDMA queue
  -> amdgpu GPU VM
  -> GPU compute/copy engine
```

Radeon Open Compute 生态里常同时出现两类 fd：

- render node，如 `/dev/dri/renderD128`：amdgpu DRM UAPI；
- `/dev/kfd`：KFD/HSA compute UAPI。

ROCm native context 必须代理 `/dev/kfd` 这一侧的 HSA/KFD 对象，因此不能只
扩展图形 native DRM context。

## 3. KFD ioctl table

[`kfd_ioctl()`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_chardev.c:3610) 用
`_IOC_NR(cmd)` 索引 `amdkfd_ioctls[]`。主要入口包括：

- `KFD_IOC_CREATE_QUEUE/DESTROY_QUEUE/UPDATE_QUEUE`
- `KFD_IOC_CREATE_EVENT/DESTROY_EVENT/SET_EVENT/RESET_EVENT/WAIT_EVENTS`
- `KFD_IOC_ALLOC_MEMORY_OF_GPU/FREE_MEMORY_OF_GPU`
- `KFD_IOC_MAP_MEMORY_TO_GPU/UNMAP_MEMORY_FROM_GPU`
- `KFD_IOC_IMPORT_DMABUF/EXPORT_DMABUF`
- `KFD_IOC_ACQUIRE_VM`
- `KFD_IOC_SVM`
- `KFD_IOC_CRIU_OP`
- debug/profiler/runtime enable

[`kfd_ioctl.h`](../../linux/include/uapi/linux/kfd_ioctl.h:56) 的 version
字段和 UAPI 注释是判断 Guest/Host 兼容性的第一部分。

## 4. Process 与 PASID

### 4.1 KFD process

[`kfd_create_process()`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_process.c:965)
以 task 的 `mm_struct` 为身份：

1. 获取并保持 `mm` 引用。
2. 等待旧的 process cleanup work 完成，处理 `exec()` 后 mm 变化。
3. 在全局 mutex 下寻找同一 `mm` 的 primary KFD process。
4. 不存在则 `create_process()`。
5. 建立 procfs/debugfs 和 IRQ drain waitqueue。

[`find_process()`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_process.c:1071)
用 `thread->mm` 找 primary process，并增加 kref。子进程或 secondary KFD
context 不能被简单视为同一 primary process。

### 4.2 Process-device 与 GPU VM

[`kfd_process_device_init_vm()`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_process.c:1847)
从 amdgpu DRM file 取得 `amdgpu_vm`，再 acquire 成 KFD/GPU VM：

- `amdgpu_amdkfd_gpuvm_acquire_process_vm()`
- reserve IB memory
- 初始化 context wave save/restore
- 保存 `pasid`

注释明确指出：IOMMU 要把 `pasid->mm` bind 到设备；process 死亡或设备移除
时才 unbind。对虚拟化而言，Host KFD process 与 Guest process 不是同一个 mm，
因此必须有额外映射和生命周期协议。

## 5. Queue 创建

### 5.1 用户参数

[`set_queue_properties_from_user()`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_chardev.c:252)
校验：

- queue percentage 和 priority；
- ring size 是 2 的幂，且有最小尺寸；
- ring、read pointer、write pointer、EOP、context save area 的 user address；
- metadata ring（AQL）；
- SDMA engine ID。

然后在 queue type 中区分：

- compute；
- compute AQL；
- SDMA；
- SDMA XGMI；
- SDMA by engine ID。

非 AQL queue 使用 PM4 packet format，AQL queue 使用 AQL format。

### 5.2 ioctl 主体

[`kfd_ioctl_create_queue()`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_chardev.c:381)：

1. 根据 user `gpu_id` 找 process-device。
2. `kfd_bind_process_to_device()`。
3. 第一次使用该 device 时分配 process doorbell slice。
4. `kfd_queue_acquire_buffers()` 把 ring/rptr/wptr/EOP/context save 等用户
   buffer 解析/映射到 GPUVM。
5. `pqm_create_queue()`。
6. 返回 queue ID。
7. 返回可 mmap 的 doorbell offset。

### 5.3 Doorbell

[`kfd_doorbell_process_slice()`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_doorbell.c:50)
决定每个 process 在 doorbell BAR 中的 slice。

[`kfd_doorbell_mmap()`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_doorbell.c:106)：

- 只允许映射完整的 single-device/single-process doorbell allocation；
- 设置 `VM_IO | VM_DONTCOPY | VM_DONTEXPAND | VM_NORESERVE |
  VM_DONTDUMP | VM_PFNMAP`；
- 使用 `pgprot_noncached()`；
- `io_remap_pfn_range()` 映射物理 doorbell aperture。

Guest HSAKMT 写 doorbell 的语义是唤醒 GPU scheduler/firmware。虚拟化后，
Host 物理 doorbell 不能直接暴露给 Guest；必须映射成 proxy doorbell 或由
Host renderer 代写。

### 5.4 PQM/DQM

[`pqm_create_queue()`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_process_queue_manager.c:324)：

1. 检查每 process 最大 queue 数；部分 ASIC 为 127 或 255。
2. 分配 queue slot。
3. 第一次 queue 时 register process 到 device queue manager。
4. MES 启用时分配 per-process context BO/index。
5. 对 SDMA/compute 分别创建 user queue。
6. 返回 process-relative doorbell offset。
7. 登记 procfs queue。

`device_queue_manager` 把用户 queue 状态放入 MQD（Memory Queue Descriptor），
scheduling firmware 在映射硬件 queue 时加载到 HQD。

## 6. Queue 执行与调度

[`amdgpu/userq.rst`](../../linux/Documentation/gpu/amdgpu/userq.rst)
给出的核心模型：

```text
UMD writes packets -> ring
UMD writes wptr -> memory
UMD rings doorbell -> firmware wakes
firmware fetches wptr -> processes ring -> updates rptr
```

硬件 queue slot 少于用户 queue 时，scheduler firmware 根据 priority/time
quantum map/unmap queue。这就是“软件/固件多路复用硬件 HQD”，不是 PCIe
SR-IOV 或 MIG。

如果发生 buffer migration：

1. preempt user queues；
2. migrate buffers；
3. update GPUVM page tables；
4. invalidate TLB；
5. resume queues。

AI compute 虚拟化必须保留这套 preemption/migration/恢复语义，否则长 kernel
可能被 Host 内存事件强制终止。

## 7. Memory allocation

### 7.1 Flags

[`kfd_ioctl_alloc_memory_of_gpu()`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_chardev.c:1124)
使用 `KFD_IOC_ALLOC_MEM_FLAGS_*`：

- `VRAM`
- `GTT`
- `USERPTR`
- `DOORBELL`
- `MMIO_REMAP`
- `WRITABLE`
- `EXECUTABLE`
- `PUBLIC`
- `AQL_QUEUE_MEM`
- `COHERENT`
- `UNCACHED`
- `EXT_COHERENT`
- `CONTIGUOUS`

### 7.2 分配流程

1. 拒绝 secondary KFD process 的 userptr。
2. SVM 开启时检查与现有 SVM interval tree 的地址冲突。
3. 按 `gpu_id` 找 process-device 并绑定。
4. Doorbell allocation 强制 page-slice 大小。
5. 通过 `amdgpu_amdkfd_gpuvm_alloc_memory_of_gpu()` 建 `kgd_mem`。
6. 在 process-device IDR 中建立 handle。
7. VRAM allocation 增加 `pdd->vram_usage`。
8. 返回 `handle` 和可 mmap offset。

userptr 的 mmap offset 字段承载 CPU user address；VRAM/GTT 则返回 mmap
offset。这是容易混淆的 ABI 细节。

### 7.3 Multi-GPU mapping

[`kfd_ioctl_map_memory_to_gpu()`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_chardev.c:1326)
接收 device ID array，对每个 peer：

1. 找 process-device；
2. bind；
3. `amdgpu_amdkfd_gpuvm_map_memory_to_gpu()`；
4. 同步 page-table update；
5. flush 各 peer TLB。

这条路径对 AI 的 P2P/多 GPU tensor 很关键。虚拟化后如果 Guest 只看到 VF 或
MIG instance，Host 物理 peer topology 与 Guest 可见 topology 可能不同。

## 8. Events

[`kfd_event_create()`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_events.c:336)
和 [`kfd_wait_on_events()`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_events.c:903)
实现 HSA event：

- signal event；
- node/device state change；
- hw exception；
- debug/profiler；
- queue event；
- memory event。

User process 通常还映射 event page，Host/GPU 可在 page 中写 signal slot。
Device interrupt 路径通过 PASID/partial event ID 找 process 和 event，
再唤醒 waitqueue。

虚拟化时完整语义包括：

- Guest event page 如何映射；
- Host GPU interrupt 如何找到 Guest event；
- partial ID 是否唯一；
- auto-reset 与并发 wait；
- queue/device reset 如何 signal；
- process teardown 如何唤醒所有 waiter。

## 9. SVM 与 page fault

### 9.1 Range 和 notifier

[`svm_range_new()`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_svm.c:336) 建立
CPU virtual-address interval range；notifier 在
[`svm_range_add_notifier_locked()`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_svm.c:110)
中注册。

[`svm_range_cpu_invalidate_pagetables()`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_svm.c:2688)
区分：

- `MMU_NOTIFY_UNMAP`：从 GPU unmap，并延迟拆分/删除 range；
- 其他 migration/invalidation：根据 XNACK/retry fault 选择 evict queue 或
  unmap range 后恢复。

### 9.2 GPU retry fault

[`svm_range_restore_pages()`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_svm.c:3075)：

1. 用 PASID 找 KFD process。
2. 找 fault 对应 node/gpu index。
3. 确认 XNACK 开启。
4. 取 process mm 并加 mmap read lock。
5. 在 SVM interval tree 找/创建 range。
6. 检查 VMA 权限。
7. `hmm_range_fault()` 获取/迁移 pages。
8. DMA map 并更新 GPU PTE。
9. 处理 HMM seqno retry、evict/migration race。

### 9.3 CPU 与 GPU 的地址一致性

[`gpusvm.rst`](../../linux/Documentation/gpu/rfc/gpusvm.rst:4) 强调：

- CPU VA 可以随时变化，physical page 才相对稳定；
- migrate_to_ram 只用 core MM page migration/locking；
- eviction 是从 GPU 回 CPU、没有虚拟地址的数据迁移；
- GPU page table invalidation 需要 notifier 提供 GPU VA；
- races 通过 retry loop，而不是一层全局 driver lock。

在 Guest/Host 跨 VM 场景，最难的是把“Guest process mm change”传播成 Host
GPU PTE invalidate/rebind，同时保证 Host 不会继续 DMA 到已释放页。

## 10. Long-running compute

[`drm-compute.rst`](../../linux/Documentation/gpu/drm-compute.rst) 把超过
10 秒的 workload 单独讨论：

- 不一定有可用 dma-fence；
- 硬件可能不可抢占或不可 swap；
- pin 全部 buffer 会带来内存 DoS；
- 需要不 eviction 的策略或 cgroup-aware accounting；
- preemption 和 recoverable page fault 是避免强制终止的关键。

AI 训练大 kernel、长 collective 或 checkpoint 都可能触发类似问题。虚拟化
层若只支持“可快速完成的图形 job”，会把 AI job 变成不可调度或易被 reset。

## 11. 虚拟化映射

| KFD 对象 | 裸机位置 | virtio/ROCm proposal 需要 |
| --- | --- | --- |
| KFD process | Host `/dev/kfd` fd | Guest/Host process 映射和 PASID |
| queue | ring/rptr/wptr/context save | proxy create/update/destroy |
| doorbell | PCI MMIO aperture | 不可直接暴露 Host BAR；代写/映射 |
| event | Host event page + IRQ | Guest event page、event ID、fence |
| GEM/BO | amdgpu GPUVM | blob/dma-buf/VA proxy |
| userptr | HMM/pin | Guest page pin、DMA、coherence |
| SVM range | CPU mm + GPU PTE | 跨 VM fault/notifier/rebuild |
| profiler/debug | ptrace + KFD | 受权限和安全策略限制 |

virglrenderer ROCm proposal 通过 HSAKMT-oriented commands 代理这些对象，最终
调用 Host libhsakmt/KFD。它解决的是“Guest ROCm runtime 使用 Host compute
device”的软件边界，不等于 VFIO、SR-IOV 或 MIG。

## 12. 故障定位

### Queue 创建失败

检查：

- Guest 是否以正确 `gpu_id` 绑定；
- ring/rptr/wptr 是否可访问；
- queue 数是否超限；
- doorbell slice 是否分配；
- AQL/SDMA type 和 engine ID；
- Guest/Host UAPI version。

### Kernel 提交后无完成

区分：

- doorbell 是否到达 Host;
- scheduler firmware 是否 map queue；
- GPU page fault；
- event page 是否可见；
- Host fence 是否回写；
- Guest event/waitqueue 是否唤醒。

### 计算读到旧数据

检查：

- userptr/pinned page cache policy；
- HMM/CPU invalidation；
- DMA map coherence；
- GPU PTE MTYPE/SNOOPED；
- migration/eviction；
- queue preemption 和 resume。

### AI job 被 kill

检查长 compute 路径是否：

- pin 过多；
- 被 GPU memory eviction；
- queue preemption 失败；
- device reset；
- fenced/unrecoverable page fault；
- process mm teardown。

## 13. 源码阅读顺序

1. `include/uapi/linux/kfd_ioctl.h`
2. `kfd_chardev.c` ioctl table 和每个 handler
3. `kfd_process.c` process/PASID/VM
4. `kfd_process_queue_manager.c` + `kfd_queue.c`
5. `kfd_doorbell.c`
6. `kfd_events.c`
7. `kfd_svm.c`
8. `Documentation/gpu/drm-compute.rst`
9. `Documentation/gpu/rfc/gpusvm.rst`
10. `virglrenderer` ROCm MR 和 `libhsakmt`

## 14. 检查题

1. 为什么 KFD process 以 `mm_struct` 而不是 PID 为主要身份？
2. PASID 在 GPU page fault、queue 和 IOMMU 中分别有什么作用？
3. queue create 为什么需要 ring、rptr、wptr、EOP、context save 多个 buffer？
4. Guest 写 doorbell 为什么不能直接落到 Host GPU MMIO BAR？
5. MQD、HQD、queue slot 和 firmware scheduler 的关系是什么？
6. userptr allocation 的 `mmap_offset` 为什么承载 CPU 地址？
7. Map memory to multiple GPUs 为什么要 flush 每个 peer 的 TLB？
8. event page 和 event waitqueue 在跨 VM 时如何对应？
9. SVM range 的 CPU invalidation 和 GPU retry fault 如何互相恢复？
10. 为什么长 compute 不能简单依赖 dma-fence 和可 evict BO？
