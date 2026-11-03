# AI GPU 计算栈与虚拟化边界

## 1. 为什么 AI 需要不同于图形虚拟化的视角

图形 workload 通常围绕 frame、draw、pipeline 和 present；AI workload
围绕模型权重、张量、kernel、长生命周期显存、梯度/优化器状态、KV cache、
checkpoint 和跨 GPU collective。

AI 系统的关键指标也不是帧率：

- prefill/decode tokens/s；
- 每 token 延迟及 P50/P99；
- 训练 samples/s 或 step time；
- HBM bandwidth、计算利用率；
- collective bus bandwidth 和尾延迟；
- 扩展效率、checkpoint 暂停时间；
- 每租户显存/算力配额。

因此不能因为 Venus/GFXStream 能执行 Vulkan compute，就推断它们可以替代
CUDA/HIP/ROCm 生态。

## 2. AI 软件到硬件的完整栈

```text
AI framework / serving stack
  PyTorch, JAX, TensorFlow, vLLM, SGLang, ...
  -> compiled graph / eager kernels
  -> CUDA runtime + cuBLAS/cuDNN/NCCL ...
     or HIP/ROCr + rocBLAS/MIOpen/RCCL ...
  -> userspace driver/runtime
     libcuda / HIP runtime / ROCr / libhsakmt
  -> kernel driver/UAPI
     NVIDIA KMD / amdgpu + KFD / Xe / others
  -> command queue and doorbell
  -> GPU copy/compute engines
  -> HBM/VRAM and peer interconnect
```

NVIDIA 与 AMD 的具体库名不同，但抽象问题相同：框架生成 kernel，
userspace runtime 管理对象和 command，kernel driver 建立 GPU VM/queue/event，
硬件执行 compute/copy。

## 3. AMD ROCm/KFD 源码视角

AMD 的 ROCm 路径不应只看 amdgpu 图形 DRM UAPI。KFD 提供 compute 相关对象。

[`kfd_ioctl.h`](../../linux/include/uapi/linux/kfd_ioctl.h:56) 中可见：

- compute/AQL/SDMA queue create/destroy/update；
- doorbell offset；
- queue priority/percentage；
- synchronous/asynchronous event 和 wait；
- VRAM/GTT/userptr/doorbell memory；
- coherent/noncoherent/uncached memory flags；
- dma-buf import/export；
- SVM attributes、preferred/prefetch location 和 access policy；
- checkpoint/restore（CRIU）与 process snapshot；
- debug/profiler 接口。

核心源码目录：

- [`kfd_chardev.c`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_chardev.c)：
  ioctl 入口。
- [`kfd_process.c`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_process.c)：
  process/PASID/device 关联。
- [`kfd_queue.c`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_queue.c)：
  queue 生命周期。
- [`kfd_doorbell.c`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_doorbell.c)：
  doorbell mapping。
- [`kfd_device_queue_manager.c`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_device_queue_manager.c)：
  queue 调度和管理。
- [`kfd_events.c`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_events.c)：
  event/signal/wait。
- [`kfd_svm.c`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_svm.c)：
  HMM/SVM range、page fault、migration。

ROCm native context 需要虚拟化的不是普通图形 draw，而是：

```text
HSA process
  + queue/AQL ring/doorbell
  + event
  + VRAM/GTT/userptr allocation
  + GPU VA/SVM range
  + fence/error state
```

## 4. AI 内存模型

### 4.1 显存不等于普通 RAM

| 存储 | 特征 | AI 影响 |
| --- | --- | --- |
| VRAM/HBM | GPU 高带宽，容量有限 | 权重、activation、KV cache 和梯度 |
| Host pinned memory | 可 DMA，长期驻留 | 数据流水、host-visible buffer |
| Pageable memory | 可换页/迁移 | 需 staging 或 HMM/SVM |
| GTT/system aperture | GPU 可访问系统内存 | overflow、host-mapped tensor |
| GPU VA | GPU 页表地址空间 | kernel/buffer/image 引用 |
| userptr/SVM | CPU/GPU 共享虚拟地址 | 少显式拷贝，但 coherence 复杂 |

### 4.2 生命周期

AI workload 常见的长期对象：

- 模型参数；
- 优化器状态；
- 梯度；
- 激活值；
- KV cache；
- CUDA graph/command capture；
- collectives communicator；
- compiled kernels 和 workspace。

图形系统的“每帧 resource 创建/释放”不是 AI 主要模式。AI 更常见的是：

```text
alloc once
  -> reuse many iterations
  -> synchronize frequently or asynchronously
  -> checkpoint/migrate rarely
  -> free at process end
```

所以虚拟化必须支持长生命周期 pin/mapping、有效 CUDA graph/queue state、
高密度小 kernel 和大块 HBM 分配。

### 4.3 Unified memory / SVM / HMM

[`gpusvm.rst`](../../linux/Documentation/gpu/rfc/gpusvm.rst:4) 描述了 GPU
共享虚拟地址、migration/eviction、fault handling 和 userptr 共同实现方向。
[`hmm.rst`](../../linux/Documentation/mm/hmm.rst) 则解释 CPU 页表和设备页表
之间的镜像/迁移机制。

虚拟化场景增加一层问题：

- Guest CPU VA 如何变成 Host GPU 可访问的 DMA 地址；
- MMU notifier/page fault 在哪一侧处理；
- 页被迁移、回收、munmap 时，Host GPU 的旧地址何时失效；
- migration 后的 Guest page table 和 GPU PTE 如何重建。

这就是 userptr 不能只依赖一个 `pin_user_pages()` 的原因。

完整的地址域、DMA mapping、coherence、migration、eviction、GPU TLB
invalidation 和各虚拟化机制的显存边界见
[GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)。

## 5. AI 计算队列、同步和错误

queue 提交、engine 调度、preemption、fence 与 reset 的硬件执行模型见
[GPU 执行模型、队列、调度与故障隔离](gpu-execution-model.md)。

### 5.1 Queue

compute workload 使用 compute/AQL/SDMA/copy queue。Doorbell 用于通知硬件；
queue memory 和 ring 通常长期存在。

虚拟化时需要区分：

- Guest 自己的 queue memory；
- Host renderer 为 Guest 创建的 proxy queue；
- Host 物理 GPU 的实际 queue；
- ring/doorbell 的共享地址和写权限。

### 5.2 同步

AI 常见同步：

- kernel completion；
- HIP stream/CUDA stream dependency；
- event；
- semaphore/timeline；
- collective operation；
- host/device memory visibility。

高吞吐训练常让多个 kernel 异步排队；推理 serving 则更关注每 token 尾延迟。
如果虚拟化的 fence 每次都触发进程/VM 往返，性能会明显受损。

### 5.3 错误

AI 的 device lost、ECC、Xid、GPU page fault、queue reset 都可能让整个训练
job 退出。虚拟化要回答：

- 故障只影响一个租户还是共享 GPU；
- reset 能否不破坏其他 context；
- 未完成 collective 如何取消；
- checkpoint 是否已保存；
- 迁移/重试能否恢复。

设备 reset、suspend/resume、snapshot、workload checkpoint 和 live migration
的状态差异见
[GPU VM 生命周期、热插拔、Checkpoint 与迁移](vm-lifecycle-migration.md)。

## 6. 训练、微调与推理的虚拟化敏感点

| 阶段 | 主要瓶颈 | 对虚拟化最敏感的部分 |
| --- | --- | --- |
| 数据预处理 | CPU、存储、数据加载 | Guest CPU、共享内存和数据搬运 |
| 模型装载 | HBM、PCIe/NVLink | 大块权重传输、page fault、内存 quota |
| Forward/Backward | compute、HBM | kernel launch、queue、tensor 生命周期 |
| 优化器 step | compute、collectives | 多 GPU 同步、P2P/网络转发 |
| 分布式训练 | NCCL/RCCL collective | topology、P2P、IOMMU、虚拟网络 |
| 微调 | HBM、checkpoint | 长生命周期模型、中断和恢复 |
| 推理 prefill | compute | 大 batch、HBM bandwidth |
| 推理 decode | 小 kernel、KV cache | 低延迟、频繁同步、显存碎片 |
| Checkpoint | 存储、HBM/PCIe | 暂停、迁移、内存快照 |

decode 阶段尤其不适合用“单次大 matrix multiplication 接近原生”来证明整体
性能；小 kernel、调度和 host round-trip 可能占主导。

## 7. virtio-gpu 如何接入 AI compute

### 7.1 图形 remoting 不能直接完成 CUDA/ROCm

VirGL、Venus 和 GFXStream 主要虚拟化图形 API/命令：

- VirGL：OpenGL；
- Venus：Vulkan；
- GFXStream：多 API streaming；
- Vulkan compute 可以承载部分 ML kernel，但不等于完整 CUDA/HIP 运行时。

AI 框架大量使用厂商库、CUDA graph、NCCL/RCCL、SVM、进程上下文和私有扩展。
只提供 Vulkan device 无法保证这些库兼容。

API、binary ABI、driver UAPI 和 GPU ISA 的完整兼容层次见
[AI Runtime、API 与虚拟化兼容性](ai-runtime-api-compatibility.md)。

### 7.2 ROCm native context 的定位

AMD 的提案通过新的 ROCm capset/驱动代理，把 HSAKMT-oriented commands 送到
Host renderer，再由 Host libhsakmt/KFD 执行。

```text
Guest HIP/ROCm
  -> Guest HSAKMT frontend
  -> virtio-gpu ROCm commands
  -> QEMU
  -> virglrenderer
  -> Host libhsakmt
  -> Host /dev/kfd + amdgpu
  -> physical GPU compute engine
```

价值：

- Guest 不直接拥有物理 PCI GPU；
- 多 VM 可共享 Host GPU compute capacity；
- 有机会保留 ROCm runtime/库兼容性；
- 可与 dma-buf/userptr/blob 共享 Guest memory。

代价：

- HSA/KFD/DRM SVM 版本强耦合；
- userptr/SVM coherence 与 lifetime；
- queue/event/doorbell 代理的延迟；
- Host GPU fault 和 reset 隔离；
- NCCL/RCCL P2P 和 topology 虚拟化；
- 全栈 ABI 尚未稳定合入。

### 7.3 当前提案与生产 AI 的差距

截止 2026-10-29，userptr/ROCm 仍处于 RFC/MR 阶段，且存在跨组件 ABI 差异。
因此不能用它替代以下已成熟路径：

- 单 VM VFIO 整 GPU；
- SR-IOV VF（硬件和驱动支持时）；
- MIG 分区加平台 assignment；
- NVIDIA vGPU 产品栈。

它代表“低层 compute UAPI remoting 让 virtio-gpu 共享 Host compute”的方向，
还不是可直接部署的完整 AI 方案。

## 8. 多 GPU AI：不能只看单卡虚拟化

一个训练 job 可能涉及：

```text
GPU0 GPU1 GPU2 GPU3
  |    |    |    |
 NVLink / PCIe / fabric
  |    |    |    |
 NCCL / RCCL collectives
```

虚拟化必须说明：

- Guest 内能否看到正确的 GPU topology；
- P2P 是否可用，是否被 IOMMU/PCIe ACS 阻止；
- VF/MIG instance 之间是否支持所需通信；
- NCCL/RCCL 是否把虚拟设备识别为可 P2P peer；
- network collective 是否经过虚拟交换机；
- 多 VF/MIG/VFIO group 是否属于同一 VM；
- collective 的尾延迟和故障恢复。

单 GPU microbenchmark 无法证明分布式训练能力。

## 9. 共享机制与 AI 适用性

| 机制 | AI 能力 | 主要优势 | 主要限制 |
| --- | --- | --- | --- |
| API remoting | 取决于 API 和厂商库 | 多 VM 共享 renderer | CUDA/ROCm 兼容与同步成本 |
| ROCm native context | Guest ROCm/HIP 代理 | Host compute 共享、较完整栈方向 | 未合入、ABI/coherence 复杂 |
| VFIO whole GPU | 完整厂商 runtime | 原生训练/推理兼容性最好 | 一 VM 独占、密度低、迁移难 |
| SR-IOV VF | Guest 厂商 driver | 多 VM、硬件调度和 quota | profile/驱动/迁移支持 |
| MIG | 硬件 compute/HBM partition | 隔离和确定性 | profile 限制、VM 平台支持 |
| vGPU | 厂商 VM 产品 | 管理和支持矩阵 | 许可、硬件、功能限制 |
| MPS/time slicing | 多进程共享同一 GPU | 容器/AI serving 简单共享 | 不是强 VM 隔离，干扰/安全风险 |

MPS 和 time slicing 是 Host 进程共享机制，不能与 SR-IOV/MIG 的硬件
隔离等同；在 K8s 中尤其容易被错误标签化。

## 10. AI 场景选择框架

### 10.1 单任务、最大兼容性

优先：

- VFIO whole GPU；
- 独立物理 GPU。

前提：能接受一 VM/容器独占，Host 可失去设备。

### 10.2 多租 VM、完整 AI runtime

优先：

- SR-IOV VF；
- vendor vGPU；
- MIG + supported VM assignment；
- 条件允许时再评估 ROCm native context。

必须核对 GPU 型号、profile、Guest driver、PF driver、Hypervisor 和 collective
支持。

### 10.3 中小模型/边缘推理、共享 GPU

候选：

- API remoting/native context（如果 runtime 兼容）；
- vGPU；
- SR-IOV；
- 同 Host 多进程 MPS/time slicing（若威胁模型允许）。

VM、容器、Kubernetes 分配层和真实隔离层的区别见
[云原生 AI 中的 GPU 共享与虚拟化](cloud-native-gpu-sharing.md)。

### 10.4 高密度推理、KV cache

重点测：

- HBM quota 和 page fault；
- 长短上下文切换；
- 每 token 延迟 P99；
- 小 kernel 调度；
- 多租户 cache 隔离。

## 11. 安全性与可观测性

AI 虚拟化需要保护模型权重、训练数据、prompt、KV cache 和 embedding。

完整威胁模型、信任边界、IOMMU/PCIe/VF/MIG 分析和安全审查表见
[GPU 虚拟化的隔离、安全与故障域](gpu-virtualization-security.md)。

需要重点审查：

- GPU 内存残余和跨租户泄漏；
- timing side channel；
- Host driver/renderer 攻击面；
- device reset/error 对其他租户的影响；
- checkpoint 与迁移数据。

机密 AI 还需研究 GPU confidential computing、encrypted memory、attestation
和 bounce-buffer 泄漏。仅靠 IOMMU 不代表模型权重在 Host 侧机密。

测量应覆盖 kernel launch、HBM bandwidth、compute utilization、first-token
latency、tokens/s P50/P99、显存碎片、page fault、migration、collective
bandwidth/latency 和租户干扰。

跨层计时、trace correlation、基准设计和瓶颈判定树见
[GPU 虚拟化性能分析与可观测性](gpu-performance-observability.md)。

## 12. 常见错误

1. 把 Vulkan compute 可用等同于 CUDA/ROCm 生态可迁移。
2. 把 GPU 显存总量等同于每个 Guest 可以安全使用的 HBM quota。
3. 把 API remoting 的官方吞吐图当作 decode 小 kernel 性能。
4. 把 MPS/time slicing 当作硬件隔离。
5. 把 MIG instance 当作完整 VM GPU。
6. 只测单 GPU，不测 NCCL/RCCL collective。
7. 只测 steady-state，不测 cold start、模型加载和 KV cache 增长。
8. 只看平均 tokens/s，不看 P99 和租户干扰。

## 13. 源码与资料下一步

基础文档：

- [Userptr/ROCm compute](../virtio-gpu/userptr-rocm-compute.md)
- [GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)
- [GPU 执行模型、队列、调度与故障隔离](gpu-execution-model.md)
- [GPU 虚拟化的隔离、安全与故障域](gpu-virtualization-security.md)
- [GPU VM 生命周期、热插拔、Checkpoint 与迁移](vm-lifecycle-migration.md)
- [GPU 虚拟化性能分析与可观测性](gpu-performance-observability.md)
- [云原生 AI 中的 GPU 共享与虚拟化](cloud-native-gpu-sharing.md)
- [AI Runtime、API 与虚拟化兼容性](ai-runtime-api-compatibility.md)
- [ROCm compute 源码跟踪](../virtio-gpu/rocm-source-trace.md)
- [GPU 虚拟化平台比较](../virtio-gpu/platform-comparison.md)
- [AI GPU 学习路线](learning-roadmap.md)
- [NCCL/RCCL Collective 源码跟踪](nccl-rccl-source-trace.md)

Linux 源码入口：

- `drivers/gpu/drm/amd/amdkfd/kfd_chardev.c`
- `drivers/gpu/drm/amd/amdkfd/kfd_process.c`
- `drivers/gpu/drm/amd/amdkfd/kfd_queue.c`
- `drivers/gpu/drm/amd/amdkfd/kfd_doorbell.c`
- `drivers/gpu/drm/amd/amdkfd/kfd_events.c`
- `drivers/gpu/drm/amd/amdkfd/kfd_svm.c`
- `include/uapi/linux/kfd_ioctl.h`
- `Documentation/gpu/drm-compute.rst`
- `Documentation/gpu/rfc/gpusvm.rst`
- `Documentation/mm/hmm.rst`

通信层的 topology、transport 和 ring/tree 细节见
[NCCL/RCCL Collective 源码跟踪](nccl-rccl-source-trace.md)。框架层后续
仍应比对 CUDA/HIP runtime、PyTorch allocator、CUDA graph、KV cache 和
分布式 checkpoint/恢复。
