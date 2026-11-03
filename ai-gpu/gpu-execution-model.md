# GPU 执行模型、队列、调度与故障隔离

## 1. 文档定位

[GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md) 解释了 GPU work
“访问什么”；本文解释 GPU work “如何被提交、排序、调度、执行和完成”。

掌握本文后，应能把任意一次 draw、copy、Vulkan dispatch、HIP kernel 或
collective 操作拆成：

```text
API object
  -> command encoding
  -> submission / queue
  -> hardware or firmware scheduling
  -> engine execution
  -> completion
  -> synchronization and error propagation
```

本文用 Linux/AMD 的公开 UAPI 和源码作为具体实例，但先给出厂商中立的模型。
NVIDIA、Intel 等实现使用不同术语，核心分层通常仍是 command processor、
queue、context、engine、memory system 和 completion。

源码基线：

- Linux `6b4b543fa18757297ac756a6fe6e64f506b7844a`
- QEMU `d73149283205645625bfa9fe029cfb68e4c4a9ae`
- Mesa `ec7b2f322abbcd593d600a14afe2d2773512af4e`
- virglrenderer `e0a1caf3b147bb26dfe524dbb85ded1177e7e49e`

本文不运行 VM、GPU workload、benchmark 或故障实验。

## 2. 一套硬件中立的执行图

```text
Application / framework
  |
  | API calls, command buffer recording, kernel launch
  v
User-mode driver / runtime
  |
  | command buffers, submission, memory references
  v
Kernel driver / VMM proxy
  |
  | queue creation, address-space validation, dependencies
  v
Submission doorbell or control queue
  |
  v
Scheduler
  | hardware scheduler, firmware scheduler, DRM scheduler, or proxy
  v
GPU engine
  | command processor / front end
  | shader array, copy engine, video engine, display engine ...
  v
Memory hierarchy
  | caches, GPU MMU/TLB, GTT, VRAM/HBM
  v
Completion
  | writeback, interrupt, timestamp, fence, event
```

这条图中每一层都可能被虚拟化：

| 层 | 直接映射时 | 软件 mediation 时 |
| --- | --- | --- |
| API | Guest runtime 直接调 Guest driver | Host renderer 代理 API |
| Command | Guest driver 编码硬件/固件命令 | Guest 编码 remote protocol |
| Queue | Guest 写真实 ring/doorbell | Guest 写 ring/blob/control queue |
| Scheduler | GPU/PF 调度 Guest queue | Host renderer/KFD/PF 代为调度 |
| Memory | Guest GPU page table + IOMMU | resource/blob/dma-buf + Host GPU VM |
| Completion | Guest 处理真实 IRQ/event | Host 转成 proxy event/virtio fence |

虚拟化并没有消灭这些层，只是把某些层的执行位置和所有权移到了 Host。

## 3. 不要混淆的五类硬件资源

### 3.1 Command front end

GPU 需要一个组件读取 command buffer、解析 packet，并把工作分发给具体 engine。
它可能叫 command processor（CP）、front end、packet processor，或位于调度
firmware 后面。

front end 的职责通常包括：

- 检查命令 packet；
- 切换/绑定 context；
- 把 state 配置给 shader/copy/media engine；
- 发起 memory read/write；
- 写 completion、timestamp 或 fence；
- 触发 page fault、invalid command 或 timeout。

在 direct assignment 中，Guest driver 产生的 packet 到达真实 front end。在
API remoting 中，Guest packet 先被 Host decoder 转成 Host API/driver 操作，
再由 Host 产生真实 packet。

### 3.2 Execution engine

“GPU”不是单一 engine。常见类别：

| engine | 典型工作 | AI 相关性 |
| --- | --- | --- |
| 3D/graphics | draw、raster、texture、shader | 渲染、部分通用 shader |
| compute | grid/workgroup、AQL/dispatch | GEMM、attention、elementwise |
| copy/DMA | GPU-GPU、RAM-GPU copy | H2D/D2H、tensor staging、checkpoint |
| video | encode/decode、format conversion | 视频推理、数据预处理 |
| display | scanout、plane composition | 桌面、远程显示 |
| RDMA/network | 部分集成的 data movement | 特定 AI fabric/DPU |

多个 engine 可以并行。一个 API queue 也不一定永久绑定一个物理 engine。

### 3.3 Shader array

shader 核心执行 wave/warp/thread 等并行工作。与虚拟化直接相关的属性包括：

- occupancy；
- register file；
- L1/shared memory；
- cache/TLB；
- context save/restore 能力；
- preemption granularity；
- ECC/fault 是否可定位到 wave 或 process。

MIG、SR-IOV VF、vendor vGPU 或 MPS/time slicing 对 shader array 的共享方式
不同；看到“有独立 queue”不代表 shader 核心已硬件分区。

### 3.4 Memory system

本文不重复完整地址模型，只强调执行相关部分：

- command fetch 需要 GPU VA；
- descriptor 和 kernel argument 可能再指向更多 GPU VA；
- GPU MMU 产生 TLB 查询；
- cache miss 可能访问 L2、VRAM/HBM、GTT、peer memory 或系统内存；
- page fault 可能阻塞 wave 或触发 retry；
- TLB invalidate 通常需要 context/range 和同步屏障。

命令执行正确不能只看 queue 是否推进；descriptor、PTE、cache 任一环节错误
都可能让 GPU 工作失败。

### 3.5 Completion machinery

常见 completion：

- engine 写 memory value；
- timestamp query；
- fence/timeline signal；
- event page update；
- interrupt/MSI-X；
- device exception，如 ECC、page fault、hang、reset。

“完成通知到达”与“全部内存副作用全局可见”仍不是同一件事，后者还依赖
memory ordering 和 API 的完成语义。

## 4. Work、submission、queue、context 和 fence

| 概念 | 含义 |
| --- | --- |
| Work | 一次 draw/dispatch/copy/kernel 等逻辑工作 |
| Command buffer | CPU 或驱动生成的命令序列 |
| Submission | 把一组 command/work 作为一个有依赖关系的批次提交 |
| Queue | 有顺序保证的提交通道，通常由 ring、wptr/rptr、doorbell 表示 |
| Engine | 执行某类 work 的硬件单元 |
| Context | GPU 状态、GPU VM、page table、scheduler 身份等集合 |
| Process/PASID | 把 device access 与某个 CPU 进程/地址空间关联 |
| Timeline | 有单调 seqno/value 的完成顺序 |
| Fence | 对 timeline 上某一点的完成/可见性承诺 |
| Job | 驱动调度器中的可调度执行单元 |

不同 API 的边界不同：

- OpenGL 可以隐式驱动提交；
- Vulkan command buffer/queue submit 显式描述依赖；
- CUDA stream 提供有序执行队列；
- HIP stream/AQL queue 语义类似；
- HSA AQL 直接暴露用户态 queue；
- VirGL/GFXStream 在 Guest/Host 之间再包一层协议。

架构讨论必须先说明使用的是哪一个“queue”，否则容易把 Vulkan queue、
virtqueue、ring buffer、硬件 HQD 和 Host runtime stream 混为一谈。

## 5. 命令编码层次

### 5.1 高层 API

应用使用 GL/Vulkan/HIP/CUDA API。此层关注 pipeline/descriptor/buffer/kernel
参数，不规定 packet bit layout。

### 5.2 API command stream

Venus 编码 Vulkan protocol command；GFXStream 编码 API stream；VirGL 编码
Gallium/GL command。Guest KMD 通常不理解这些语义，只负责：

- 复制或引用 command bytes；
- 关联 resource BO；
- 提交到 virtqueue；
- 建立 fence；
- 处理 completion。

### 5.3 Driver/firmware packet

Host renderer 最终调用 Host Vulkan/OpenGL/native driver。真实 driver 再生成
PM4、AQL 或厂商 packet 给 GPU/firmware。

AMD 的公开模型可在
[`amdgpu/userq.rst`](../../linux/Documentation/gpu/amdgpu/userq.rst:15) 中看到：
UMD 把 IP-specific packet 写入 ring，更新 wptr，然后写 doorbell；firmware
读取 wptr 并处理 packet。

### 5.4 为什么层次不能混用

以下说法都不准确：

- “VirGL command 就是 GPU 硬件 packet”；
- “Venus ring head 就是 Vulkan fence”；
- “GFXStream 的 ASG ring 是 GPU command processor ring”；
- “virtio vring 是 GPU 的 AQL queue”；
- “写 doorbell 就等于 kernel 已完成”。

每一层有自己的格式、生命周期和 completion 语义。

## 6. Guest 与 Host 的提交路径

### 6.1 VirGL

```text
Guest Mesa draw
  -> VirGL command buffer
  -> DRM_VIRTGPU_EXECBUFFER
  -> virtio control virtqueue
  -> QEMU
  -> virglrenderer/vrend
  -> Host OpenGL
  -> Host driver packet
  -> physical GPU
```

Guest KMD 搬运 opaque command。Host vrend 解码 draw、维护 GL state，再生成
Host GL submission。Guest 可见 fence 来自 Host GL completion → renderer
callback → QEMU fence response → Guest dma_fence。

### 6.2 Venus

```text
Guest Vulkan encoder
  -> shared Venus ring
  -> notify / SUBMIT_3D
  -> QEMU proxy
  -> render server
  -> vkr dispatch
  -> Host Vulkan queue
  -> Host GPU
```

Venus ring head 前移表示 Host 已消费 wire command，不表示 Host VkFence 已
signal。Vulkan queue/fence/semaphore/barrier 仍由 Host Vulkan 和 GPU 完成。

### 6.3 GFXStream/Rutabaga

```text
Guest GFXStream ICD
  -> ResourceTracker/VkEncoder
  -> ASG ring
  -> virtio-gpu
  -> QEMU Rutabaga
  -> C FFI / Rust component
  -> libgfxstream_backend
  -> Host Vulkan or native backend
```

GFXStream 的 ring seqno、backend fence 和 QEMU virtio response 是三层不同
completion。Rutabaga 只负责把请求跨 C/Rust/component 边界转发。

### 6.4 Virtio-GPU native DRM context

Guest 侧提交更接近 DRM UAPI，Host native renderer 维护 Host GPU VM、BO、
queue timeline。它减少了 API 语义差距，但仍由 Host native driver 执行，不等同
VFIO passthrough。

### 6.5 ROCm/KFD native compute

AMD KFD 暴露：

- compute/AQL queue；
- SDMA/XGMI copy queue；
- ring、rptr、wptr、EOP、context save area；
- process-relative doorbell；
- event page；
- GPU VM/PASID。

KFD UAPI 的
[`kfd_ioctl_create_queue_args`](../../linux/include/uapi/linux/kfd_ioctl.h:73)
一次性描述这些资源。ROCm native context 提案要代理的是整套 queue/event/
doorbell/VM 语义，而不是只把一次 HIP kernel launch 编成 API command。

## 7. Queue 的典型实现

### 7.1 Ring、指针和门铃

```text
producer:
  write packet at write index
  publish write pointer / tail
  ring doorbell

consumer:
  wake scheduler/firmware
  fetch write pointer
  read packets
  execute or dispatch work
  publish read pointer / head
  signal completion as appropriate
```

[`amdgpu/userq.rst`](../../linux/Documentation/gpu/amdgpu/userq.rst:17) 明确
把 queue 描述为带 rptr/wptr 的 ring。现代 AMD GPU 的用户 queue 还包含
MQD/HQD、context save area、EOP buffer 和 doorbell。

### 7.2 Doorbell

doorbell 通常是 GPU 的 MMIO/BAR aperture 中一块非 cacheable write 区域。
写 doorbell 只是 wakeup/update 信号：

- 不等于 packet 已执行；
- 不等于 rptr 已推进；
- 不等于 fence 已 signal；
- 不保证 queue 已映射到硬件 slot。

KFD 的
[`kfd_doorbell_mmap()`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_doorbell.c:106)
使用 noncached 映射和 `io_remap_pfn_range()`。Virtio-GPU/ROCm mediation 中，
Host physical doorbell 不能直接暴露给不受信任 Guest，通常需要 proxy 或 Host
代为写。

### 7.3 聚合门铃

[`amdgpu/userq.rst`](../../linux/Documentation/gpu/amdgpu/userq.rst:79) 描述
了 oversubscription 场景：当用户 queue 多于硬件 queue slot，某些 queue 可能
未映射，直接 ring doorbell 可能丢失。部分硬件支持未映射 queue 的 doorbell，
部分需要 aggregated doorbell 唤醒 scheduler。

这说明“每次 write doorbell 都立即启动目标 queue”不是通用假设。

### 7.4 Shadow pointer

让 firmware 读取用户控制的 wptr memory 有验证/顺序问题。实现可能使用 shadow
pointer、受限写权限或受保护 fence packet。AMD user queue 文档中的 protected
fence packet 只允许特权实体写 completion memory，用户 queue 只能触发/读取
受控结果。

虚拟化代理必须保持同类安全属性，不能把 Host fence memory 任意暴露给 Guest。

## 8. Context、process、address space 与 PASID

### 8.1 Context 不是单一对象

“context”可能指：

- graphics API context；
- Vulkan device/queue；
- GPU scheduler context；
- GPU VM/address space；
- renderer context ID；
- virtio context ID；
- KFD process/queue context；
- 硬件 context-save slot。

讨论 context switch 时必须说明切换的是 GL state、command stream、GPU page
table、scheduler slot，还是完整硬件 context。

### 8.2 PASID

PASID 把 device memory access 与进程地址空间关联，并帮助 GPU fault/IOMMU
定位 process。Linux KFD process 的 device VM 保存 PASID；event interrupt
也通过 PASID 等身份找到对应 process。

在虚拟化中：

- Guest PASID 不能直接当作 Host PASID；
- Host allocator 需要单独的 PASID/VM；
- GPU fault 可能先在 Host 发生，再转换成 Guest event；
- process exit 必须解除 GPU/IOMMU 绑定并 drain fault。

### 8.3 GPU VM

GPU VM 通常包含：

- page table root；
- PTE/PDE；
- TLB tagging；
- BO/resource mappings；
- fault/retry 策略；
- queue/ring/context-save backing。

VFIO Guest 驱动直接管理 Guest GPU VM；API remoting 则由 Host renderer 管理
Host GPU VM。两者都可以让 Guest 应用使用 GPU VA，但 PTE 和 fault 所有者不同。

## 9. Scheduler：谁决定下一步运行谁

### 9.1 调度层

可能出现多个 scheduler：

| 调度器 | 负责 |
| --- | --- |
| Guest GPU scheduler | Guest context/queue 的选择 |
| Host KFD/MES/firmware | Host physical queue/context 调度 |
| DRM scheduler | kernel job 排序和 fence 依赖 |
| PF scheduler | SR-IOV VF 之间时间片/配额 |
| Hypervisor/VMM scheduler | vCPU、device thread、IRQ thread |
| Host OS scheduler | QEMU/renderer/用户态服务线程 |

一个 Guest submission 可能先经过 Guest scheduler，再经过 Host runtime 和物理
GPU scheduler。任何一层排队都可能造成尾延迟。

### 9.2 AMD MQD/HQD 模型

[`amdgpu/userq.rst`](../../linux/Documentation/gpu/amdgpu/userq.rst:29)
给出具体例子：

- 每个用户 queue 有 MQD；
- MQD 保存在 GPU accessible memory；
- scheduling firmware 把 MQD 加载到 HQD；
- run list 决定哪些 queue 可调度；
- 用户 queue 数可超过 HQD slot；
- firmware 按 priority/time quantum map/unmap queue。

这证明“软件创建了 1000 个 queue”不等于“硬件同时执行 1000 个 queue”。

### 9.3 优先级与公平性

queue priority 是调度提示，不等于绝对资源上限。生产方案还要定义：

- priority aging/starvation；
- compute percentage；
- HBM bandwidth quota；
- queue count cap；
- preemption timeout；
- per-tenant error containment；
- CPU/GPU cgroup accounting。

一个线程有高 priority 不意味着它绕过 IOMMU、memory quota 或 Host scheduler。

### 9.4 图形和 AI 的调度差异

图形 workload 常以 frame 为 natural preemption boundary；AI 可能有：

- 极短的小 kernel；
- 多个 stream 的并发 kernel；
- 长时间运行的 matrix kernel；
- collective 的同步等待；
- graph replay 的低 CPU 开销要求。

因此不能把图形路径的 fence round-trip 和 reset 策略直接套到 AI。

## 10. 同步层级

### 10.1 五个层级

一次 GPU 提交至少可能涉及：

1. **CPU 指令顺序**：应用线程按程序顺序写 command/memory。
2. **CPU memory ordering**：release/acquire 或 barrier 使数据对消费者可见。
3. **queue ordering**：同一 queue 内 work 的顺序。
4. **inter-queue dependency**：semaphore/timeline/syncobj 建立跨 queue 依赖。
5. **completion visibility**：fence/event/interrupt 表示 work 完成，并配合
   cache flush/API barrier 使用。

缺少任一层都可能表现为“偶发读到旧数据”。

DRM binary/timeline syncobj、handle/fd/sync_file 和 eventfd 的逐函数语义见
[DRM Syncobj 与 Timeline 同步源码跟踪](drm-syncobj-source-trace.md)。

### 10.2 Ring head/tail 的语义

Venus/GFXStream 的环：

```text
Guest publishes tail after writing command
Host consumes commands and publishes head
```

head 的推进通常表示 command stream 已被读取/解析，而不是 GPU kernel 已完成。
真正的 GPU completion 必须看 Host API fence、native timeline 或 event。

### 10.3 Virtio fence

[`virtio_gpu_fence_emit()`](../../linux/drivers/gpu/drm/virtio/virtgpu_fence.c:84)
给 command header 写 `VIRTIO_GPU_FLAG_FENCE`、`fence_id` 和可选 `ring_idx`。
Guest 端再把它映射为 `dma_fence`。

[`virtio_gpu_fence_event_process()`](../../linux/drivers/gpu/drm/virtio/virtgpu_fence.c:110)
只在 `dma_fence` context 匹配时，signal 指定 fence 以及同 context 中更早的
fence。这解释了为什么 `ring_idx`/fence context 不能随意合并。

### 10.4 QEMU command completion

[`virtio_gpu_process_cmdq()`](../../qemu/hw/display/virtio-gpu.c:1253) 处理
QEMU command queue，并可能：

- renderer blocked 时暂停；
- command 可 suspend；
- 带 fence 的 command 移入 fence queue；
- Host callback 完成后写 virtio response。

Host 完成路径可以来自 renderer 线程，QEMU 需要安全地把完成转回 AIO/设备线程。

### 10.5 层级示例

一次 Vulkan compute dispatch：

```text
Guest records dispatch
  -> submits Vulkan command buffer to Venus/GFXStream ring
  -> Host creates/submits Host Vulkan work
  -> GPU command processor runs shader/dispatch
  -> Host VkFence signal
  -> renderer callback
  -> QEMU response
  -> Guest VkFence signal
```

这里至少有“Host 收到 command”“Host queue 接受 submission”“GPU 完成 dispatch”
三个不同进度点。

## 11. Preemption 与 context switch

### 11.1 Preemption 粒度

从轻到重可能是：

- wave/thread block 边界；
- draw/dispatch/job 边界；
- queue 边界；
- process/context 边界；
- engine reset；
- whole-GPU reset。

硬件越难在细粒度抢占，越依赖内存 pin 或长 job 限制。

### 11.2 迁移与 preemption

AMD user queue 文档说明，buffer migration 前会 preempt user queues、迁移
buffer、更新 GPUVM PTE、invalidate TLB，再 resume queue。

这个顺序意味着：

```text
residency change
  -> stop old access
  -> update PTE
  -> invalidate stale translation
  -> resume
```

只做 PTE 写入而没有 GPU TLB invalidate，GPU 仍可能命中旧翻译。

### 11.3 Long-running compute

[`drm-compute.rst`](../../linux/Documentation/gpu/drm-compute.rst:1) 指出：

- 长 compute 可能无法在 10 秒内完成；
- 某些硬件不能 preempt 或 swap out；
- 无 recoverable fault 时，unbind 可能只能强制终止 workload；
- pin 全部内存会形成 DoS；
- cgroup-aware accounting/eviction 是可能的控制手段。

因此虚拟化层需要明确：

- job 最大运行时间；
- preemption 是否可用；
- page fault 是否可恢复；
- reset 的 owner；
- timeout 后是 queue reset 还是 device reset；
- checkpoint 能否承担恢复。

### 11.4 Context save area

硬件 context switch/preemption 常需要 driver 预分配 context save buffer。
KFD queue creation 接收 `ctx_save_restore_address/size` 和 `ctl_stack_size`。
VF/vGPU 若把 queue 直通给 Guest，必须保证这些 buffer 的 GPU VA、可访问性和
Host reset 路径一致。

## 12. Fault、hung queue 与 reset

### 12.1 故障分类

| 故障 | 典型范围 | 首要证据 |
| --- | --- | --- |
| invalid command | queue/context | Host parser/driver error |
| GPU page fault | process/context | GPU fault record、VMA/PTE |
| queue hang | queue/engine | wptr/rptr、timeout、EOP |
| fence timeout | submission/timeline | completion path、IRQ/event |
| ECC/poison | memory range/device | RAS event、memory exception |
| engine reset | engine/affected contexts | reset event、queue status |
| whole-GPU reset | device/all tenants | PCIe/reset status、VM loss |
| device lost | entire VM/device | AER/FLR/reset completion |

名称相似但影响范围可能完全不同。

### 12.2 Queue reset

[`amdgpu/userq.rst`](../../linux/Documentation/gpu/amdgpu/userq.rst:137) 说明
许多 GFX/compute/SDMA queue 可单独 reset。hung queue 可能在 preemption 失败时
被发现；driver 查询失败 preempt 的 queue 并 reset/记录状态。

这不等于所有 GPU 或所有虚拟化方案都支持 per-queue reset。API remoting 通常
依赖 Host driver 的 reset 能力，VFIO 则可能触发 PF/device 级恢复。

### 12.3 Reset 为什么影响内存

reset 可能需要：

- 停止所有 queue；
- 撤销 GPU VM/page table；
- 丢弃 VRAM 或 context save 数据；
- 重新初始化 firmware/engine；
- 重建 queue/event/fence；
- 让 Guest 观察 `MEMORY_LOST` 或等价错误。

KFD hardware exception data 中有 `reset_type/reset_cause/memory_lost`，说明
reset 与内存丢失是应用/UAPI 必须看到的语义，而不只是内核内部清理。

### 12.4 多租户边界

必须区分：

- 某 Guest context 报错；
- 某 Guest queue hang；
- 某 VF 被 reset；
- 某 MIG instance 受影响；
- 整块 GPU 被 reset。

## 13. 虚拟化机制如何改变执行边界

### 13.1 软件 API remoting

代表路径：VirGL、Venus、GFXStream。

改变点：

- Guest 不直接驱动物理 GPU；
- Guest command 需要 Host parser；
- Host runtime 重建 GPU queue/context；
- fence 需要跨 Guest/Host 转换；
- Guest reset 和 Host reset 不是同一个动作。

优势：

- 可跨厂商共享 Host GPU；
- Guest 不需要匹配物理 PCI GPU；
- 可用 API 层做 capability 裁剪；
- 更容易把多个 VM 放进同一 Host renderer 调度。

代价：

- API/版本兼容矩阵大；
- parser 是安全攻击面；
- submit/fence 有 Host round-trip；
- 私有 CUDA/HIP 扩展难覆盖；
- Guest 和 Host 的 queue/fault 对象难一一映射。

### 13.2 Virtio-GPU native DRM context

改变点：

- Guest UMD 看到接近原生的 DRM ioctl/context；
- Host renderer 维护 native GPU driver 的 VM、BO、queue 和 fence；
- Virtio 仍承担控制和资源 transport。

它缩小了 API 语义差距，但没有让 Guest 拥有真实 PCI GPU。GPU fault、reset、
scheduler 和 Host heap 仍由 Host 侧管理。

### 13.3 VFIO

改变点：

- Guest 原生 driver 直接写物理 GPU MMIO；
- Guest 原生 queue/doorbell/IRQ 语义基本保留；
- Host IOMMU 约束 DMA；
- VMM 转发 MSI-X、配置空间和部分平台事件。

优势：

- Guest 软件栈兼容性最高；
- Host 不解析图形/compute command；
- 性能路径接近原生。

代价：

- 通常整 GPU 独占；
- IOMMU group/ACS/FLR/reset 复杂；
- VRAM/context 迁移困难；
- Host 失去设备；
- Guest kernel/firmware 漏洞可能攻击 GPU/平台。

### 13.4 SR-IOV VF

改变点：

- Guest 原生 driver 驱动一个 VF；
- 每个 VF 有独立 PCI function；
- PF 和硬件 scheduler 管理 VF/context/VRAM；
- Guest queue 与物理 GPU scheduler 的边界仍由 VF 硬件定义。

“有独立 VF”比软件 API remoting 的 PCIe 边界强，但不自动意味着：

- 每个 VF 有完全独立 engine；
- 每个 VF 有固定 HBM bandwidth；
- error/reset 完全隔离；
- 支持 live migration；
- P2P 一定可用。

### 13.5 mdev 与 MPT

mdev 提供 mediated device 的 Host 框架；MPT 是更宽泛的半直通设计。

执行路径可能组合：

- Guest 处理部分 queue/doorbell；
- Hypervisor/Host 截获敏感 MMIO；
- Host scheduler 代管 engine；
- Host driver 映射 Guest doorbell 或转发 submission；
- Guest IRQ 由 event/fence 转换而来。

判断 MPT 的关键不是名称，而是哪些寄存器/队列/内存窗口直通，哪些被 trap。

### 13.6 MIG

MIG 在物理 GPU 内切分 shader/L2/HBM 等资源。它能提供硬件资源分区，但与
PCIe function 和 Guest driver 暴露方式是两个维度。

将 MIG instance 放入 VM 需要 platform assignment 机制；它是否拥有独立
queue/context/reset 边界取决于具体 GPU 和软件实现，不能只从“MIG”名称推断。

### 13.7 Vendor vGPU

vendor vGPU 通常组合：

- Guest 厂商 driver；
- 虚拟 PCI function/aperture；
- Host scheduler 和 time slicing；
- VRAM/context 管理；
- 可能的硬件 partition/SR-IOV；
- vGPU manager 和迁移接口。

vGPU 的 queue 提交可能保留 Guest driver 语义，但最终仍由 Host manager 调度。
安全性和性能必须按具体产品、profile、driver 和 hypervisor 版本验证。

### 13.8 横向执行矩阵

| 方案 | 谁编码真实 GPU packet | 谁调度物理 engine | Guest 是否处理真实 IRQ |
| --- | --- | --- | --- |
| VirGL | Host OpenGL driver | Host GPU/driver | 否，经 virtio fence |
| Venus | Host Vulkan driver | Host GPU/driver | 否，经 proxy/timeline |
| GFXStream | Host backend/driver | Host GPU/driver | 否，经 component/QEMU |
| native DRM context | Host native driver | Host GPU/driver | 间接 |
| VFIO | Guest GPU driver | GPU/firmware | 是，经 VMM |
| SR-IOV VF | Guest VF driver | PF/GPU scheduler | VF/driver 路径 |
| mdev/MPT | 依直通边界 | Host 或硬件 scheduler | 依设计 |
| MIG | Guest/Host 取决于 assignment | GPU partition scheduler | 依 VM 暴露方式 |
| vGPU | Guest driver + Host backend | Host vGPU scheduler | 依产品 |

## 14. AI workload 如何放大执行层差异

### 14.1 小 kernel 与 submit rate

推理 decode、elementwise、reduce、sampling 等阶段可能产生很多短 kernel。
每次提交若经历：

```text
user runtime -> ioctl -> virtqueue -> QEMU -> renderer -> Host runtime -> GPU
```

则会放大小 kernel 的固定 overhead。

评估时应区分：

- API/runtime 编码；
- submit syscall/ioctl；
- virtqueue kick 与通知；
- Host parser；
- Host runtime launch；
- GPU queue wait；
- kernel execution；
- fence wakeup。

只看 GPU utilization 可能把 launch-bound 错判成 GPU-bound。

### 14.2 CUDA graph / graph replay

graph replay 的目标之一是减少重复 kernel launch 的 CPU 成本。虚拟化方案
需要保留：

- command graph 中的 pointer/backing 稳定；
- queue/event 对象有效；
- graph capture/replay 语义；
- graph 更新与内存迁移的协调；
- graph 内多 kernel 和多 stream 依赖。

如果每次 replay 都被 Host proxy 重新展开或触发大量 fence 往返，graph 收益会
被吃掉。

### 14.3 Prefill 与 decode

| 阶段 | 计算特点 | 执行层敏感点 |
| --- | --- | --- |
| prefill | 大矩阵、高并行、sequence 长 | HBM bandwidth、shader occupancy |
| decode | batch 小、kernel 短、频繁同步 | launch latency、queue wait、P99 |
| sampling | 小规模控制/随机逻辑 | Host-device round-trip |

同一虚拟化方案可能 prefill 接近原生，但 decode 明显退化。

### 14.4 训练 kernel 与 collective

训练 kernel 往往运行时间长，HBM/collective 占主导。此时 API remoting 的
单次 submit overhead 可能不是主要瓶颈，但：

- collective 的同步等待会暴露尾延迟；
- 内存 fault/migration 会打断长 kernel；
- queue reset/device lost 会摧毁整个 step；
- P2P/topology 错误会改变 collective 算法。

### 14.5 Host 与 device 的流水线

高吞吐推理需要持续供给 GPU。可用简化模型表示：

```text
throughput = work_batch / max(cpu_submit_time,
                              gpu_execution_time,
                              host_proxy_time,
                              memory_time)
```

如果 GPU execution 是 100 us，而 Host submission/proxy 总成本是 300 us，
GPU 只会周期性空闲。增加 batch 可摊薄 submit 成本，但会增加首 token 延迟和
显存占用。

### 14.6 多租户干扰

即使每个 Guest 有自己的 queue，共享 engine、L2、HBM 和 copy engine 仍可能
互相影响：

- 高优先级 compute 挤压低优先级推理；
- 大量 copy 占用 HBM bandwidth；
- 一个长 kernel 阻止 preemption；
- 一个租户的 page fault 造成 scheduler stall；
- reset 中断同 engine 的其他 context。

因此要同时记录 queue latency、GPU engine utilization、HBM bandwidth 和
tenant-level completion。

## 15. 性能分解

### 15.1 端到端延迟

一次完整 submission 可拆为：

```text
T_end_to_end
  = T_application
  + T_encode
  + T_submit
  + T_transport
  + T_host_decode
  + T_queue_wait
  + T_engine_execution
  + T_memory_stall
  + T_completion_transport
  + T_wakeup
```

各路径不一定会出现所有项：

- VFIO 通常没有 API decode；
- Venus/GFXStream 有 ring 和 Host decode；
- VirGL submit 次数由 command buffer flush 决定；
- ROCm context 还会经历 libhsakmt/KFD proxy；
- 反馈等待可能由 polling/eventfd/interrupt 决定。

### 15.2 CPU 与 GPU 时间不能简单相加

异步 workload 中 CPU 可在 GPU 执行时继续提交后续 work。总吞吐取决于流水线
瓶颈，而不是每个 kernel 的 CPU 时间与 GPU 时间相加。

可能出现：

- CPU-bound：submit rate 低，GPU 有空洞；
- GPU-bound：GPU 持续忙，CPU 等待队列；
- fence-bound：每步都同步，无法形成深度；
- copy-bound：频繁 H2D/D2H 或 peer copy；
- memory-bound：HBM/L2/GTT bandwidth；
- scheduler-bound：queue 太多、优先级反转、preemption 太频繁。

### 15.3 Submit amplification

常见放大器：

- 每个小 draw/dispatch 都 ioctl；
- 每个 command 都要求 fence；
- ring notify 过于频繁；
- Host renderer 每 submit 都创建临时对象；
- QEMU 复制大 payload；
- fence callback 造成线程切换；
- 过多 queue 争抢少量 HQD slot。

代码审查时先数“每个 workload 单元产生多少跨边界事件”，再讨论 GPU FLOPs。

### 15.4 Queue depth 与背压

深 queue 隐藏 CPU 和网络延迟，但带来：

- 内存占用；
- cancellation 困难；
- timeout 归因困难；
- priority inversion；
- 用户误以为 submit 即完成。

浅 queue 降低尾延迟和资源占用，但更容易暴露 Host round-trip。

推理 serving 和多租训练往往需要不同 queue depth 策略。

## 16. 可观测性：先定位层级

### 16.1 Virtio-GPU 信号

可利用或检查：

- Guest `virtio_gpu` trace events；
- command header 的 `type/flags/fence_id/ctx_id/ring_idx`；
- control virtqueue 的 avail/used；
- QEMU `cmdq/fenceq/renderer_blocked`；
- output fence FD 和 `dma_fence` timeline；
- QEMU 已有的 `virtio-gpu` trace events。

Linux
[`virtgpu_trace.h`](../../linux/drivers/gpu/drm/virtio/virtgpu_trace.h:20)
的记录包含 fence ID、context、queue 和空闲 descriptor 数，适合先判断问题在
virtio transport 还是更下层。

### 16.2 Vulkan ring 信号

Venus/GFXStream 要看：

- ring tail 是否发布；
- Host head 是否推进；
- ring 是否长期满；
- proxy submit 是否返回；
- Host object 是否已创建；
- Host Vulkan fence 是否 signal；
- QEMU fence response 是否写回。

ring head 滞留通常说明 Host parser/dispatch 没有消费；head 前进但 fence 不
完成则更可能是 Host GPU/runtime。

### 16.3 KFD/ROCm 信号

可用证据包括：

- KFD queue 的 ring/rptr/wptr；
- doorbell 是否到达 Host；
- MQD/HQD/reset 状态；
- event page 和 waitqueue；
- SVM fault/migration SMI 事件；
- queue/device reset 事件；
- GPUVM PTE/TLB 状态。

[`kfd_ioctl.h`](../../linux/include/uapi/linux/kfd_ioctl.h:531) 定义了 VM
fault、migration、queue eviction 等 SMI event，可用于建立 Host 侧时间线。

### 16.4 硬件计数器

GPU utilization、SM/CU busy、HBM bandwidth、L2 hit rate、copy engine busy、
power/clock 可以回答“GPU 是否真的忙”。它们不能直接回答：

- Guest command 是否到 Host；
- fence 是否被错过；
- 哪个 proxy 队列在等待；
- PTE 是否失效；
- 哪个租户造成干扰。

因此硬件计数器要与虚拟化层事件关联。

### 16.5 证据链最小集合

单次请求或 kernel 至少应有：

```text
timestamp:
  guest API entry
  guest submit
  virtqueue used
  host parser begin/end
  host GPU submit
  queue/engine start
  completion/fence
  guest wakeup
```

不要用单个“平均 GPU 利用率”解释端到端尾延迟。

## 17. 常见错误结论

### “ring tail 已更新，所以 work 已执行”

错。tail 只发布 command 数据；consumer 仍可能未读取或未执行。

### “head 已推进，所以 Vulkan fence 应完成”

错。head 通常只表示 Host 消费 wire command。

### “doorbell 写成功，所以 queue 已调度”

错。doorbell 只唤醒/signals scheduler；queue 可能未映射、被抢占或尚未轮转。

### “VF 独立，所以 kernel 不会互相干扰”

错。VF 可能共享 HBM bandwidth、engine 时间片、fabric、firmware 和 reset 域。

### “MIG 独立，所以无需虚拟化层”

错。MIG 提供硬件资源分区，VM ABI、PCIe exposure、IRQ、迁移和管理仍由平台
软件完成。

### “有 recoverable page fault，所以可以随时抢占”

错。recoverable fault 解决缺页恢复，不自动提供 kernel preemption、TLB
同步或 context save 能力。

### “fence 已完成，所有数据自动全局一致”

错。fence 完成只覆盖其定义的作用域；cache 属性、barrier、view 和 memory
ordering 仍需满足。

## 18. 架构评审模板

### 18.1 Workload

```text
API/runtime:
graphics / Vulkan compute / HIP / CUDA / collective

Work shape:
large kernel / many small kernels / copy-heavy / sync-heavy

Latency goal:
throughput / P50 / P99 / first token / step time
```

### 18.2 Execution path

```text
Command encoder:
Guest UMD / Guest driver / Host decoder

Submit mechanism:
ioctl / virtqueue / shared ring / doorbell / trap

Queue owner:
Guest / Host renderer / PF / firmware / vendor manager

Scheduler:
software / firmware / hardware / multiple levels

Engine partition:
shared / time sliced / VF / MIG / vGPU partition
```

### 18.3 Synchronization

```text
Intra-queue ordering:
Inter-queue dependency:
Host completion object:
Virtio/proxy fence:
CPU wakeup:
Timeout/cancel:
```

### 18.4 Failure

```text
Invalid command scope:
Page fault scope:
Queue reset scope:
Engine reset scope:
GPU reset scope:
Memory loss semantics:
Tenant recovery:
```

### 18.5 Performance

```text
Cross-boundary events per work unit:
Payload bytes per event:
Queue depth:
Fence frequency:
Expected idle gaps:
Tenant interference:
```

只有这些字段明确后，才能比较“接近原生”“低延迟”或“高密度”。

## 19. 检查题

1. command front end、engine、shader array、GPU MMU 和 completion 各自做什么？
2. 为什么 work、submission、queue、context 和 fence 不能互换？
3. VirGL command、Venus wire command、PM4/AQL packet 分别属于哪一层？
4. Guest KMD 在 VirGL/Venus/GFXStream 中为什么不需要理解 API 语义？
5. virtio vring 与 GPU queue 的 wptr/rptr/doorbell 有何区别？
6. doorbell 写成功为什么不能证明 queue 已运行？
7. ring head、Host VkFence、Guest dma_fence 分别证明什么？
8. AMD MQD 与 HQD 的关系是什么，为什么用户 queue 数可大于硬件 slot？
9. aggregated doorbell 解决什么问题？
10. Guest PASID 为什么不能直接复用为 Host PASID？
11. preemption 粒度有哪些，为什么长 compute 的风险不同？
12. buffer migration 为什么要 preempt、更新 PTE、invalidate TLB 后 resume？
13. invalid command、page fault、queue hang、engine reset、GPU reset 的影响范围
    有何不同？
14. VFIO、SR-IOV、MIG 和 vGPU 分别如何改变“谁执行真实 packet、谁调度 engine”？
15. 为什么 decode 可能比 prefill 更受虚拟化 submit 路径影响？
16. 为什么 GPU utilization 不能替代 guest-to-host latency 证据？

## 20. 与其他专题的关系

- 地址、residency、coherence：  
  [GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)
- AMD queue/doorbell/event/SVM 源码：  
  [AMD KFD/ROCr 源码跟踪](amdkfd-source-trace.md)
- Virtio-GPU 四路径源码：  
  [四路径源码横向对照](../virtio-gpu/source-trace-cross-path.md)
- 硬件/软件/API 方案比较：  
  [GPU 虚拟化平台比较](../virtio-gpu/platform-comparison.md)
- 多 GPU collective：  
  [NCCL/RCCL Collective 源码跟踪](nccl-rccl-source-trace.md)
