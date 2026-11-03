# GPU VM 生命周期、热插拔、Checkpoint 与迁移

## 1. 文档定位

静态语义回答“正常路径如何工作”；生命周期回答“运行中的设备如何停止、
保存、恢复、替换或失败”。后者决定一个 GPU 虚拟化方案能否用于生产。

本文区分：

- cold boot / warm reset / FLR；
- suspend/resume；
- hibernation；
- VM snapshot；
- process checkpoint/restore；
- workload checkpoint；
- VFIO device state migration；
- VM live migration；
- GPU hotplug/reassignment。

这些机制中任何一个成功，都不能自动证明其他机制可用。

## 2. 七种“保存/恢复”不是同义词

| 机制 | 主要动作 | 是否保持 Guest 连续运行 | 是否保存 GPU 完整状态 |
| --- | --- | --- | --- |
| Queue/device reset | 丢弃或重建 engine/context | 否 | 否 |
| Suspend/resume | 暂停设备、保存驱动必要状态 | 短暂停 | 依驱动，不保证透明 |
| Hibernation | 写入系统镜像后关机 | 否 | 依设备和驱动 |
| VM snapshot | 保存 VMM/Guest 可见状态与内存 | 可短暂停顿 | 取决于设备 migration |
| Process CRIU | 冻结进程对象并 restore | 进程停 | 只要 UAPI 支持的对象 |
| Workload checkpoint | App 主动保存 tensor/step | App 决定 | 通常不含硬件 context |
| Live migration | 源 VM 运行期间复制并切换 | 设计目标是不中断 | 仅当所有设备可迁移 |

“VM 内存可复制”只解决 CPU 虚拟地址中的字节；“GPU queue/context/VRAM
可复制”是额外问题。

## 3. GPU 状态分类

### 3.1 Guest 可见设备状态

- virtio/PCI config；
- feature negotiation；
- BAR/MMIO aperture；
- MSI-X/IRQ routing；
- capset/protocol version；
- output/scanout 配置。

### 3.2 Transport 状态

- virtqueue 地址、index、avail/used ring；
- inflight descriptors；
- control/cursor/event queue；
- 未完成 response；
- 共享 ring 的 head/tail/status。

### 3.3 对象 namespace

- Guest context/resource ID；
- Host renderer context/resource table；
- dma-buf/opaque fd；
- queue ID、event ID、timeline ID；
- range/BO handle。

### 3.4 内存状态

- Guest RAM/GPA mapping；
- resource backing 内容；
- VRAM/HBM residency；
- GTT/system memory mapping；
- GPU page tables；
- context save/MQD/EOP/ring buffer；
- dirty page 信息。

### 3.5 执行状态

- queue wptr/rptr；
- scheduler run list；
- current job/wave；
- preemption/context save；
- copy/media engine state。

### 3.6 同步状态

- fence seqno/timeline value；
- semaphore/syncobj；
- event page 和 waitqueue；
- interrupt pending mask；
- display acquire/release fence。

### 3.7 Host 可重建状态

有些对象可不保存，只需通过 Guest-visible metadata 在目的地重建：

- Host Vulkan/GL device；
- shader/pipeline cache；
- Host renderer context；
- 可能的 queue；
- 临时 workspace。

前提是重建结果、资源 ID 和 fence 语义对 Guest 等价。

## 4. 标准生命周期状态机

```text
CREATED
  -> CONFIGURED
  -> RUNNING
  -> QUIESCING
  -> SAVING / STOP_COPY
  -> TRANSFER
  -> DESTINATION_LOAD
  -> RESUMING
  -> RUNNING

failure at any transition
  -> ROLLBACK or ABORT
  -> RESET/REBUILD or DEVICE_LOST
```

每一步必须明确：

- 哪些 queue 允许继续提交；
- 哪些 completion 可以到达；
- memory dirty tracking 是否开启；
- 是否允许 destination 提前创建对象；
- 何时 source 失去 ownership；
- 失败后哪一侧恢复运行；
- Guest 是否看到 error/reset。

## 5. 初始化与设备发现

### 5.1 Virtio-GPU

Guest 启动时：

1. 枚举 virtio PCI/MMIO 设备；
2. feature negotiation；
3. 读取 config/capsets；
4. 建立 control/cursor virtqueue；
5. UMD 查询 capset 和 params；
6. context init；
7. resource/blob/ring 创建。

迁移到不同 Host 时，Guest 不应突然看到不兼容的 feature/capset。热升级若改变
capset，可能需要 Guest driver reset 或冷启动，而不是透明迁移。

### 5.2 VFIO/SR-IOV/vGPU

Guest 依赖：

- PCI vendor/device/subsystem ID；
- BAR size/位置；
- PCIe capability/ROM；
- link state；
- MSI-X 数量；
- VF/PF/profile identity；
- firmware/device version。

目的地设备若无法保持等价 ABI，Guest 驱动可能拒绝 probe 或错误初始化。

## 6. Virtio-GPU 当前源码能证明什么

### 6.1 2D VMState

QEMU
[`virtio_gpu_save()`](../../qemu/hw/display/virtio-gpu.c:1442) 会遍历有
`pixman image` 的 resource，保存：

- resource ID、尺寸和格式；
- iov 地址/长度；
- pixman image 的像素数据；
- scanout 配置。

[`virtio_gpu_load()`](../../qemu/hw/display/virtio-gpu.c:1506) 重建 image、
恢复 iov mapping 和 scanout。

这只说明 2D 软件设备状态有 save/load 代码。它不证明 3D renderer、GPU
context、VRAM 或 dma-buf 状态可迁移。

### 6.2 Blob VMState 与 blocker

QEMU 有 `vmstate_virtio_gpu_blob_state`，在 blob feature 开启时保存部分 blob
设备状态。但源码同时存在：

- VirGL 配置注册 `"virgl is not yet migratable"` blocker；
- Rutabaga 注册 `"rutabaga is not yet migratable"` blocker；
- 超大 blob 可触发 migration blocker。

因此必须逐配置判断：

```text
virtio-gpu 2D
virtio-gpu + blob
virtio-gpu-gl/VirGL
virtio-gpu-rutabaga/GFXStream
virtio-gpu Venus with proxy renderer
```

不能从一个 variant 的 VMState 外推到所有 variant。

### 6.3 Reset

[`virtio_gpu_reset()`](../../qemu/hw/display/virtio-gpu.c:1848)：

- 必要时通过 bottom half 在安全上下文执行；
- destroy resources；
- 清除 scanout surface；
- 丢弃 `cmdq` 中的 command 并 detach virtqueue element；
- 丢弃 `fenceq` 中未完成 command；
- 最后 reset virtio base。

这说明 reset 是 destructive recovery，不是 migration。未完成 fence 会被清除，
Guest driver 必须能处理 device reset 语义。

### 6.4 Async command/resource 操作

VirGL 的 blob unmap/unref 可能 suspend 或异步完成。reset 路径需要处理：

- renderer reset；
- async fence queue；
- scanout；
- 被挂起的 QEMU command；
- 已开始的 mapping teardown。

如果 reset 与 callback/unmap 竞态未定义，迁移 blocker 反而是在防止错误状态
被保存。

## 7. API Remoting 的迁移难点

### 7.1 Guest 状态并不完整

Guest 只知道 protocol object ID、ring 内容和 fence。Host 可能仍持有：

- VkDevice/VkQueue；
- GL context；
- pipeline/shader cache；
- resource backing；
- dma-buf export；
- proxy socket/timeline shmem；
- pending GPU work。

### 7.2 Host object 可否重建

可重建对象需要：

- 等价 API/version；
- 兼容 GPU/driver；
- 可恢复的 descriptor/pipeline；
- stable resource identity；
- fence value 映射；
- renderer 能重放必要的 lazy state。

不可重建对象则需要 Host state serialization，而 API 不一定提供。

### 7.3 Pending work

迁移前通常必须：

1. 停止新 submit；
2. 等 Host work 完成或 preempt；
3. drain fence/event callback；
4. 保存 Guest/Host object 关系；
5. 使旧 Host mapping 失效；
6. 在目的地重建并恢复。

只暂停 vCPU 不能保证 Host renderer 的异步线程和 GPU work 已停止。

### 7.4 External resource

以下对象跨越 Host 外部：

- dma-buf/opaque fd；
- Wayland/X11 display；
- VFIO/network device；
- shared memory；
- checkpoint storage；
- peer GPU。

它们可能需要独立 migration/同步协议，而不能塞入普通 RAM 复制。

## 8. VFIO 有状态迁移

### 8.1 UAPI 状态机

Linux VFIO migration UAPI 定义：

- `RUNNING`；
- `STOP`；
- `STOP_COPY`；
- `RUNNING_P2P`；
- `PRE_COPY` / `PRE_COPY_P2P`。

核心 feature 是
[`VFIO_DEVICE_FEATURE_MIGRATION`](../../linux/include/uapi/linux/vfio.h:1042)。

`PRE_COPY` 允许设备继续运行并跟踪内部 state 变化；`STOP_COPY` 最终冻结状态。
P2P 状态用于控制 peer DMA，避免迁移期间设备仍写旧数据。

### 8.2 Dirty tracking

迁移效率取决于：

- RAM dirty pages；
- device internal dirty bytes；
- VRAM dirty/residency；
- migration iteration 收敛速度；
- source/destination link bandwidth。

[`VFIO_MIG_GET_PRECOPY_INFO`](../../linux/include/uapi/linux/vfio.h:1258)
返回 dirty/estimate 信息。持续高速写入的 AI workload 可能一直无法收敛。

### 8.3 目的地兼容

必须验证：

- migration version；
- state region size；
- device capability；
- GPU 型号/firmware；
- Guest driver；
- IOMMU group/ACS；
- P2P 和 interrupt；
- 目的地资源容量。

“支持 VFIO migration UAPI”只表示接口存在，不表示具体 GPU driver 支持所有
状态。

## 9. SR-IOV、mdev、MPT、MIG 与 vGPU

### 9.1 SR-IOV VF

需要保存/恢复：

- VF PCI 可见状态；
- VF 内部 GPU context/VRAM；
- PF 侧 quota/profile；
- queue/reset 状态；
- P2P/network/interrupt。

若目的地没有等价 VF profile，不能直接用同一 migration state。

### 9.2 mdev/MPT

mdev 框架的 create/remove 不等于 migration。具体 vendor driver 必须提供：

- migration state；
- freeze/resume；
- dirty tracking；
- destination compatibility；
- teardown/rollback。

MPT 若把 queue/doorbell/VRAM 直通给 Guest，必须找到能与之匹配的状态保存机制。

### 9.3 MIG

需要区分：

- GPU instance/compute instance 配置；
- MIG device 到 VM 的 assignment；
- partition 内 CUDA/HIP context；
- VRAM/HBM content；
- 目的地是否支持相同 MIG profile。

MIG instance 不是可透明迁移的独立 PCIe 设备。迁移必须绑定平台、profile、
driver 和 hypervisor 支持矩阵。

### 9.4 Vendor vGPU

要检查：

- source/destination GPU 和 vGPU type；
- vGPU manager 版本；
- Guest/Host driver；
- licence/feature；
- display/remote protocol；
- VRAM/context migration；
- 操作窗口和停机时间。

厂商产品可能支持 suspend/resume、snapshot 或 live migration 的不同子集，
不能根据产品名推断。

## 10. KFD CRIU：进程 checkpoint 不是 VM migration

### 10.1 UAPI 流程

KFD
[`kfd_ioctl_criu_args`](../../linux/include/uapi/linux/kfd_ioctl.h:693) 定义：

```text
PROCESS_INFO
  -> pause execution and evict queues
  -> CHECKPOINT BOs/queues/events/SVM ranges
  -> UNPAUSE or restore
```

恢复：

```text
RESTORE
  -> RESUME
```

### 10.2 为什么这是进程级

CRIU 面向一个 Linux process 的 KFD 对象，不自动保存：

- 整个 VM 的 CPU/device state；
- 虚拟 PCI config/BAR；
- Guest page table/GPA；
- hypervisor 与 vGPU manager state；
- VMM virtqueue；
- 其他 VF/peer VM。

它适合进程检查点和部分作业恢复，不能替代 VM live migration。

### 10.3 MQD/context save

AMD KFD 的 MQD manager 提供 checkpoint/restore MQD、control stack 等实现；
hibernation resume 还可能要求 MQD BO 在新位置重新 pin。

这说明 GPU scheduler state 和 memory residency 都必须显式恢复，不能只恢复
CPU 指针。

## 11. AI workload 的 checkpoint

### 11.1 应用 checkpoint

一个训练 checkpoint 至少包括：

- model weights；
- optimizer state；
- gradient/activation 中需要跨 step 保留的部分；
- RNG；
- scheduler/step counter；
- data loader position；
- distributed communicator/collective state；
- rank/world/topology mapping。

应用级 checkpoint 可由框架主动协调，一般比透明 GPU migration 更稳健。

### 11.2 GPU 内部状态

训练框架通常不保存 shader wave/queue 的微观状态，而是在 step boundary
同步后保存逻辑状态。这降低了设备/驱动迁移要求。

反之，VM live migration 试图保存完整 Guest/device 状态，约束强得多。

### 11.3 KV cache 与推理

推理服务可能希望迁移：

- model weights；
- KV cache；
- request/scheduler state；
- sampling/RNG；
- stream/fence。

KV cache 动态且大，迁移成本可能远高于权重。可以设计：

- 不迁移 KV cache，只重建 session；
- 在 request boundary 迁移；
- prefix cache 单独复制/重建；
- 目的地保留同 profile 的空闲容量。

### 11.4 Collective

迁移一个 rank 会破坏 collective 假设：

- communicator 中 peer 连接；
- GPU P2P mapping；
- network QP；
- sequence number；
- timeout rank；
- ring/tree channel state。

因此多卡训练 live migration 通常需要框架协同，而不是只迁移单块 GPU。

## 12. Suspend/resume 与 hibernation

ECC/poison、reset cause 和 recovery scope 的深入说明见
[GPU RAS、ECC、Reset 与 AI Job 恢复](gpu-ras-ecc-reset-recovery.md)。

### 12.1 Suspend

系统 suspend 仍保留 RAM，但设备可能掉电。GPU driver 需要：

- 停止 queue；
- 保存必要 register/context state；
- 把 VRAM content 转为可恢复状态；
- 关闭 DMA/IRQ；
- resume 后重建 GPU VM/queue/TLB。

### 12.2 Hibernation

Hibernation 会把系统镜像写入持久存储并关机。GPU MQD/context BO 可能需要
重新定位/pin，event/fence 也必须恢复。

Linux KFD 源码明确存在 hibernation 相关 MQD BO 重新 pin 逻辑；这再次说明
GPU state 与普通 CPU page 的状态机不同。

### 12.3 Suspend 不等于迁移

同一 Host 上 resume 可以重新查询原物理 GPU，而迁移要求目的地兼容。两者不能
共享同一个“success”判断。

## 13. Hotplug 与重新分配

安全流程：

```text
stop new submission
  -> drain/quiesce work
  -> invalidate GPU PTE/TLB
  -> detach/unmap
  -> remove virtual device/IRQ
  -> clear VRAM/context/internal state
  -> reassign to next tenant
```

缺少任一步都可能导致：

- 旧 DMA 写新租户内存；
- 旧 event 唤醒新 device；
- 旧 VRAM 数据泄漏；
- Guest driver 将 stale completion 与 new queue 混淆。

## 14. 故障与回滚矩阵

| 阶段 | 失败示例 | 安全动作 |
| --- | --- | --- |
| Quiesce | Host GPU work 未停止 | 取消迁移或 preempt/reset |
| Save | 状态读取失败 | 保持 source running 或 restart |
| Transfer | 网络中断 | 丢弃不完整目的地状态 |
| Destination load | driver/API 不兼容 | 不切换 ownership |
| Resume | queue/fence 恢复失败 | rollback 或 device lost |
| Commit 后 source cleanup | old mapping 清理失败 | 隔离设备、阻止重分配 |

最重要的 invariant：

```text
同一时刻只能有一个 owner 可提交真实 GPU work。
```

在尚未证明 destination 接管成功前，source 不能释放唯一的恢复状态。

## 15. 迁移/恢复证据清单

未来解除“不运行实验”限制后，一个完整验证至少记录：

1. source/destination QEMU、Linux、GPU driver/firmware；
2. virtio/PCI ABI 与 feature/capset；
3. Guest device discovery 日志；
4. queue/context/resource 数量和 ID；
5. Guest RAM、VRAM、GTT、pinned memory 大小；
6. dirty bytes 和 convergence iterations；
7. pending GPU work/preemption；
8. fence/timeline/event signal；
9. display/WSI state；
10. P2P/network collective；
11. source teardown 和 destination first completion；
12. failure/rollback 结果；
13. 数据校验和；
14. 安全清理与 stale DMA/TLB 检查。

迁移成功不能只用“VM 还在运行”判断，必须证明：

- Guest workload 数据正确；
- 旧 Host 不再写 Guest memory；
- 新 Host queue/fence 正常；
- 外部资源没有指向旧设备；
- reset/rollback 不会损坏 source。

## 16. 当前方案的可迁移性判断

| 方案 | 当前可安全声明的范围 |
| --- | --- |
| Virtio-GPU 2D | QEMU 有部分 VMState，但需检查具体配置/blob |
| VirGL/virtio-gpu-gl | 当前 QEMU 明确注册 migration blocker |
| Venus/proxy renderer | 需完整 Host renderer state 与版本协议 |
| GFXStream/Rutabaga | 当前 QEMU 明确 blocker；snapshot API 不等于完成 |
| Virtio-GPU native DRM | 取决于 native renderer 与 Host driver state |
| VFIO | 只有 migration UAPI 和设备实现都支持时 |
| SR-IOV VF | 依赖 PF/VF/platform/firmware/driver 支持矩阵 |
| mdev/MPT | 完全取决于 vendor implementation |
| MIG | 依赖 assignment、profile、destination 与 vendor 工具 |
| vendor vGPU | 按产品版本配置和官方 support matrix |
| KFD CRIU | process-level checkpoint，不自动等于 VM migration |

## 17. 常见错误结论

### “virtio-gpu 有 VMState，所以 GPU VM 可迁移”

错。源码仅说明某些 2D/blob/scanout 字段有 save/load，且加速后端有 blocker。

### “Rutabaga 有 snapshot API，所以 QEMU live migration 可用”

错。QEMU device realize 仍注册 migration blocker，Host object/fd/fence 也需要
完整恢复。

### “VFIO migration feature bit 存在，所以某 GPU 可迁移”

错。必须检查具体 vfio-pci variant、GPU driver、firmware 和目的地兼容。

### “MIG instance 能分配给 VM，所以可以透明迁移”

错。分配能力和 migration state 是不同能力。

### “KFD CRIU 支持 checkpoint，所以整台 ROCm VM 可迁移”

错。CRIU 面向 process/KFD 对象，不自动覆盖 VMM、PCI、显示和跨 rank collective。

### “vGPU 可以暂停，所以可以 live migrate”

错。suspend/resume、snapshot 和 live migration 是不同状态机。

## 18. 检查题

1. Reset、suspend/resume、snapshot、process checkpoint 和 live migration 有何
   不同？
2. 迁移 GPU 设备时为什么要同时保存 Guest ABI、真实 GPU context 和 VMM state？
3. Virtio-GPU 2D VMState 与 VirGL/Rutabaga 可迁移性为什么不一致？
4. Host shader/pipeline cache 哪些可重建，哪些必须序列化？
5. 为什么暂停 vCPU 不够，还要 drain Host renderer 和 GPU work？
6. VFIO `PRE_COPY`、`STOP_COPY`、P2P 状态分别解决什么？
7. dirty bytes 为什么可能导致 migration 无法收敛？
8. KFD CRIU 的 `PROCESS_INFO/CHECKPOINT/RESTORE/RESUME` 各做什么？
9. MQD/context save 和 VRAM content 为什么必须一起考虑？
10. Hotplug 前为什么要 invalidate TLB、drain IRQ 并清数据残留？
11. 为什么 NCCL/RCCL collective 的一个 rank 不能随意迁移？
12. 迁移失败时如何保证 source/destination 不会同时成为 GPU owner？

## 19. 与其他专题的关系

- 显存、migration/eviction 和 teardown：  
  [GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)
- queue、preemption、fence 和 reset：  
  [GPU 执行模型、队列、调度与故障隔离](gpu-execution-model.md)
- 故障域、热插拔安全和迁移数据：  
  [GPU 虚拟化的隔离、安全与故障域](gpu-virtualization-security.md)
- 各硬件/软件机制：  
  [GPU 虚拟化平台比较](../virtio-gpu/platform-comparison.md)
- KFD queue/event/SVM 源码：  
  [AMD KFD/ROCr 源码跟踪](amdkfd-source-trace.md)
- Rutabaga snapshot 边界：  
  [Rutabaga 源码跟踪](../virtio-gpu/rutabaga-source-trace.md)
