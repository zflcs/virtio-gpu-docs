# AI Runtime、API 与虚拟化兼容性

## 1. 核心结论

“有 GPU 设备”不等于“完整 AI 栈可运行”。虚拟化至少可能保留五种不同契约：

1. **源码/API 兼容**：应用源码还能编译；
2. **用户态二进制 ABI**：已有 `.so`/executable 仍能加载并调用；
3. **driver UAPI 兼容**：`ioctl`、对象、queue、fence 和版本协议仍匹配；
4. **Guest/Host 设备语义兼容**：虚拟设备的 capability 与物理设备足够接近；
5. **硬件 ISA/firmware 兼容**：编译产物和驱动能在目标 GPU 执行。

设备直通通常保留完整栈。API remoting 只保留被代理的 API 子集。
Vulkan compute capability 不能自动推出 CUDA/HIP 库兼容。

OpenCL ICD、Level Zero、SYCL backend 与 oneAPI 的层次和 virtio-gpu 映射见
[OpenCL、Level Zero、SYCL 与 AI compute 虚拟化边界](opencl-levelzero-sycl-virtualization.md)。

## 2. AI 软件栈

### 2.1 NVIDIA/CUDA

公开概念路径：

```text
PyTorch / JAX / TensorFlow / custom app
  -> CUDA libraries (GEMM, DNN, communication, ...)
  -> CUDA runtime / compiled extension
  -> CUDA driver API / libcuda
  -> NVIDIA kernel driver and firmware
  -> GPU
```

NVIDIA CUDA userspace 与 driver 内部通常不是开源实现。本文只把公开 ABI、
运行时对象和产品文档作为证据，不能虚构源码函数调用链。

### 2.2 AMD/ROCm

```text
PyTorch / JAX / HIP app
  -> HIP runtime / ROCr / ROCm libraries
  -> libhsakmt
  -> /dev/kfd KFD UAPI
  -> amdgpu GPU VM/queues
  -> GPU

graphics path:
  -> Mesa/ROCm DRM client
  -> /dev/dri/renderD*
  -> amdgpu DRM UAPI
```

AMD compute 不能用“代理一个 amdgpu DRM context”完整替代，因为 ROCm 还依赖
KFD/HSA queue、event、SVM 和 process 语义。详细源码见
[AMD KFD/ROCr 源码跟踪](amdkfd-source-trace.md)。

### 2.3 Vulkan/OpenGL

```text
Application
  -> Vulkan loader / ICD
  -> Vulkan driver
  -> kernel driver / GPU
```

Venus、GFXStream 代理的是 Vulkan API/wire command；VirGL 代理 Gallium/OpenGL
命令。它们不会自动提供 CUDA runtime、HIP runtime、NCCL/RCCL 或厂商私有
driver API。

## 3. 兼容性层次

### 3.1 Source compatibility

应用重新编译后可运行：

- API 名称和语义相同；
- 所用 extension 存在；
- 编译器支持目标；
- 没有依赖物理 GPU 的私有 header。

这仍不保证已有二进制或性能模型一致。

### 3.2 Binary ABI compatibility

已有用户态库可加载：

- `.so` symbol/version 匹配；
- struct layout/alignment 一致；
- ioctl number/size/version 一致；
- handle/pointer ownership 一致；
- allocator/runtime 版本匹配。

API remoting 若只提供 Vulkan，原本加载 `libcuda`/`libhsakmt` 的二进制不会
自动工作。

### 3.3 Driver UAPI compatibility

用户态 driver 与 KMD/Host proxy 必须对以下对象有同一理解：

- device discovery；
- context/process；
- queue/ring/doorbell；
- memory allocation/import/map；
- VM/PTE；
- fence/event/syncobj；
- reset/error；
- profiling/debug。

KFD 明确公开
[`KFD_IOCTL_MAJOR_VERSION/MINOR_VERSION`](../../linux/include/uapi/linux/kfd_ioctl.h:53)。
版本号只是入口；每个 ioctl 的 struct size、flag 和新字段仍需逐项检查。

### 3.4 Hardware/ISA compatibility

- CUDA cubin 通常面向特定 GPU architecture；
- PTX 可能允许 JIT 到新架构，但仍有工具链/feature 约束；
- HIP fat binary/offload 也有目标架构；
- Vulkan SPIR-V 由 driver 编译，不保证所有 extension；
- driver/firmware 版本会影响调度、memory 和错误语义。

“源码能编译”不能替代目标 hardware/profile 的已验证支持矩阵。

## 4. CUDA 兼容的关键对象

| 对象 | 兼容问题 |
| --- | --- |
| CUDA context | Guest/Host context 身份与生命周期 |
| stream/event | queue/fence 是否保留顺序与 backend |
| module/function | cubin/PTX/compiler/runtime 是否兼容 |
| device memory | HBM quota、allocation 和 mapping |
| pinned/mapped memory | DMA/coherence 和 Host 可见性 |
| Unified Memory | fault/migration 归属 |
| CUDA Graph | 捕获/重放语义、pointer、kernel launch |
| IPC handle | 跨进程/VM 是否允许安全共享 |
| VMM/peer access | GPU VA、P2P/IOMMU/ACS 是否支持 |
| library handles | cuBLAS/cuDNN/custom library 是否只走被代理 API |

CUDA Graph 和第三方库是 API remoting 的重点：

- graph 可能捕获大量生命周期状态；
- library 可能依赖私有 driver 行为；
- IPC/VMM/P2P API 可能不能仅用高层命令重放实现。

## 5. ROCm 兼容的关键对象

| 对象 | 兼容问题 |
| --- | --- |
| KFD process/PASID | Guest mm 与 Host KFD process 映射 |
| AQL/SDMA queue | ring/wptr/rptr/doorbell 语义 |
| event | signal/wait/auto-reset/reset |
| VRAM/GTT/userptr | residency、pin、coherence |
| SVM range | CPU/GPU 同 VA、fault/migration |
| dma-buf | import/export 和 external handle |
| CRIU state | process checkpoint/restore |
| profiler/debug | 权限、版本和租户边界 |

ROCm native context 提案的价值，是争取让现有 HIP/ROCr 栈继续工作，而不是
让每个应用改写为 Vulkan compute。其当前 ABI 和社区状态见
[Virtio-GPU Userptr 与 ROCm](../virtio-gpu/userptr-rocm-compute.md)。

## 6. Vulkan 兼容的边界

Vulkan 应用需要：

- loader 找到正确 ICD；
- Vulkan core 版本满足；
- instance/device extension 完整；
- queue family 能力正确；
- memory type/format/modifier 兼容；
- timeline semaphore、external memory、WSI extension 可用；
- validation/error 行为一致。

Venus/GFXStream 是不同 wire protocol 和 Host backend，不能从一个 ICD 的
结论外推另一个。

### 6.1 Vulkan compute 不等于 CUDA/HIP

可迁移：

- 自己编写并重新编译的 SPIR-V compute kernel；
- 使用标准 Vulkan buffer/image/descriptor；
- 没有 CUDA/HIP/厂商库依赖的应用。

不能自动获得：

- CUDA runtime/driver API；
- cuBLAS/cuDNN；
- HIP/ROCr/libhsakmt；
- NCCL/RCCL；
- CUDA IPC/graph/VMM 私有语义；
- 依赖特定 GPU architecture 的编译器产物。

## 7. AI framework 兼容

### 7.1 框架不是单一 API

PyTorch/JAX/TensorFlow 等通常包含：

- Python/调度层；
- C++ dispatcher；
- C++/CUDA/HIP extension；
- BLAS/DNN/attention/communication library；
- allocator/cache；
- compiler/JIT/graph；
- profiler/debug hooks；
- distributed process group。

只代理框架常用的一个大 kernel 不能证明框架整体兼容。

### 7.2 PyTorch caching allocator

caching allocator 会：

- 大块预分配；
- 复用 tensor storage；
- 按 stream 维护 event；
- 依赖 memory stats/fragmentation；
- 与 graph capture 和 custom extension交互。

若 GPU memory 全部可由 Host renderer 透明分配，仍要保证 process/context 之间
不能错误复用，且 allocator 查询的 device capability/内存统计真实。

### 7.3 Compiled extension

自定义 CUDA/HIP extension 常直接调用：

- runtime/driver API；
- device function；
- stream/event；
- custom library。

API remoting 若不支持这些 ABI，框架可能 import 失败或 fallback，导致“模型
能加载但性能/结果来自错误路径”。

### 7.4 Distributed

NCCL/RCCL 依赖：

- device IPC/P2P；
- topology discovery；
- network plugin；
- CUDA/HIP event/stream；
- communicator init；
- multi-rank synchronization。

Vulkan device 或单卡 API remoting 不会自动提供 collective 兼容。

## 8. 虚拟化机制保留了什么契约

| 机制 | Guest 可运行的软件栈 | 保留的主要契约 | 不自动保留 |
| --- | --- | --- | --- |
| VFIO 整卡 | Guest vendor driver/runtime | PCI + KMD UAPI + 完整 GPU 行为 | 多租密度、自动迁移 |
| SR-IOV VF | Guest VF driver/runtime | VF PCI + vendor UAPI | 整卡 peer/engine/profile |
| vGPU | Guest vGPU driver | 厂商虚拟设备/驱动 ABI | 是否 live migration |
| mdev/MPT | vendor-defined guest stack | vendor-defined | 多厂商兼容、完整隔离 |
| native DRM context | driver UAPI client | 部分 native UAPI/命令 | 物理 PCI、完整 reset/迁移 |
| ROCm proxy | Guest ROCm/HIP | HSAKMT/KFD-oriented subset | 尚未稳定全栈 |
| Venus | Vulkan ICD | Vulkan wire/API subset | CUDA/HIP/NCCL |
| GFXStream | GFXStream Vulkan ICD | Vulkan/API streaming subset | 任意 CUDA/HIP 二进制 |
| VirGL | Mesa VirGL | OpenGL/Gallium subset | CUDA/HIP/native driver |
| 容器共享 Host driver | Host matching stack | Host driver/library ABI | VM 级隔离 |

## 9. ABI 兼容检查表

### 9.1 User-space

```text
Application binary:
  architecture / ABI / glibc/libstdc++

Runtime:
  CUDA runtime or HIP/ROCr

Libraries:
  BLAS/DNN/communication/custom extensions

Compiler:
  PTX/cubin/HSACO/SPIR-V/toolchain version

ICD/loader:
  Vulkan/OpenGL ICD JSON and symbol dispatch
```

### 9.2 Driver UAPI

```text
Device discovery:
  PCI ID / DRM node / UUID / capability query

Objects:
  context / queue / event / VM / BO / fence

Version:
  ioctl minor / wire protocol / capset / extension

Error semantics:
  timeout / reset / OOM / page fault / device lost
```

### 9.3 Host

```text
Physical GPU architecture:
Driver/firmware:
Libraries:
Topology/P2P/NIC:
Profile:
```

任何一层不匹配，都可能使 Guest 二进制无法加载或执行。

## 10. API Remoting 可行性审查

### 10.1 应用是否只使用被代理 API

检查：

- 动态加载了哪些 `.so`；
- 是否 dlopen CUDA/HIP/Vulkan/OpenGL；
- 是否调用 vendor private symbol；
- 是否通过 subprocess 启动 GPU 工具；
- 是否依赖 `/proc`、sysfs、device node 或 driver query。

### 10.2 对象能否跨边界表达

- pointer/handle 是否只是 Guest namespace ID；
- Host 是否需要真实 pointer；
- stream/fence/event 是否有 timeline 映射；
- external memory/dma-buf 是否可 import；
- GPU VA/VMM/peer access 是否有同等能力；
- graph/module/pipeline 是否可重建。

### 10.3 错误语义是否保留

应用通常依赖：

- `VK_ERROR_DEVICE_LOST`；
- CUDA/HIP launch/OOM/illegal address；
- stream/fence timeout；
- context reset；
- unsupported extension。

若 proxy 把所有错误折叠成 generic failure，应用可能错误重试或破坏状态。

### 10.4 性能和兼容性是否分开

一个 API remoting 方案可能：

- 功能正确但 decode 很慢；
- 大 GEMM 性能好但不支持 graph；
- 能跑某模型但不支持自定义 extension；
- 只在特定 Host Vulkan/ROCm 版本可用。

选型时需要二维矩阵：

```text
feature compatibility
  ×
performance for target workload
```

## 11. 版本与兼容性故障

### 11.1 Protocol version

Venus/GFXStream/ROCm wire protocol 需要 Guest/Host 双方理解：

- wire version；
- message size；
- enum/flag；
- object lifetime；
- error code。

版本不匹配可能表现为：

- device 创建失败；
- capset 查询成功但 submit 崩溃；
- 某 API 无条件失败；
- 功能被静默降级。

### 11.2 Driver/runtime mismatch

常见：

- Host 新 driver + Guest 旧 runtime；
- Guest 新 runtime + Host 旧 proxy；
- library 依赖新 memory/graph API；
- Vulkan ICD 暴露的 extension 与 Host 实现不一致。

从 userspace 看这是“某个 API 不可用”，根因可能在 driver UAPI 或 proxy。

### 11.3 Capability overclaim

最危险的情况：

```
advertised capability
  != actual Host support
  != safe Guest/Host ABI combination
```

应用按 advertised capability 选择 fast path，随后在首次罕见操作时失败。

### 11.4 Error propagation

例如：

```
Guest VkCreateDevice
  -> proxy creates Host context
  -> Host allocation fails
  -> renderer returns generic error
  -> Guest sees VK_ERROR_OUT_OF_DEVICE_MEMORY?
```

如果错误码改变，应用的重试和回退逻辑可能失效。

## 12. 工作负载兼容矩阵

| API/workload | 源码重编译 | 现有二进制 | 典型虚拟化 |
| --- | --- | --- | --- |
| Vulkan compute 自研 kernel | 可行 | 可行（有匹配 ICD） | Venus/GFXStream |
| OpenGL/Gallium 应用 | 可行 | 可行（Mesa） | VirGL |
| CUDA 训练框架 | 通常需要完整 CUDA 栈 | 需要 VFIO/vGPU 等 | 整卡/VF/vGPU |
| HIP/ROCm 应用 | 依赖 ROCm 栈 | 需要 KFD/HSA 兼容 | VF/SR-IOV/ROCm 提案 |
| PyTorch + 自定义 CUDA extension | 依赖 driver ABI | 很难通过高层 API remoting | 整卡/VF/vGPU |
| NCCL/RCCL collective | 依赖 GPU/network ABI | 需要完整运行时 | 整卡/VF/vGPU |
| 容器 GPU workload | 依赖 Host driver | 容器内可直接使用 | MIG/MPS/time slice/整卡 |

表中的“典型”不是对特定版本的保证。

## 13. 方案选择

### 13.1 最大兼容性

优先：

- VFIO 整卡；
- SR-IOV VF；
- vendor vGPU；
- 在 Host 上原生运行的容器。

这些路径让现有 CUDA/HIP/PyTorch/自定义 extension 按原软件栈工作，代价是
硬件隔离/密度/迁移/运维约束。

### 13.2 图形和 Vulkan compute

可评估：

- Venus；
- GFXStream；
- native DRM context；
- VirGL（OpenGL）。

先确认应用确实只用被代理 API，而不是把它误当作通用 CUDA/HIP bridge。

### 13.3 ROCm compute remoting

只有当 Linux/QEMU/renderer/ROCm 前端/发行版版本全部对齐，且目标 workload
只依赖已支持的 KFD/HSA 子集时才评估。

### 13.4 容器/Kubernetes

分配层选择 MIG/VF/MPS/time slicing 后，还要确认：

- 厂商 driver/runtime；
- GPU UUID/profile；
- device node；
- topology/P2P；
- memory/queue quotas；
- lifecycle。

编排标签不改变应用 ABI 需求。

## 14. 未来验证计划

当前不执行实验。解除限制后，按以下顺序验证：

1. **API 枚举**：列出实际 dlopen 的库和 ioctl；
2. **最小对象**：device/context/queue/memory/fence；
3. **数据正确性**：小 tensor、随机 seed、checksum；
4. **库兼容**：BLAS/DNN/custom extension；
5. **graph/IPC/external memory**；
6. **错误路径**：OOM、invalid address、device lost；
7. **性能**：launch、copy、HBM、P99；
8. **多租/collective**；
9. **reset/迁移/升级**。

每一步都应同时记录路径是否 fallback。

## 15. 常见错误结论

### “Vulkan compute 能跑矩阵乘，所以能替代 CUDA”

错。CUDA runtime、厂商库、extension、graph、IPC、NCCL 和工具链是额外契约。

### “只要 API 名称相同，二进制就兼容”

错。ABI、struct layout、driver UAPI、extension、错误码和对象生命周期同样重要。

### “ROCm 是 libdrm 上层”

错。ROCm compute 依赖 `/dev/kfd`/HSA/KFD 对象；图形路径使用 amdgpu DRM。

### “VFIO 是唯一能保证兼容的方案”

不完全。它是强原生栈路径，但实际兼容性仍取决于 Guest/Host GPU 和 driver
版本；Vulkan/ROCm 专有代理在特定子集上也可能兼容。

### “Host renderer 可以透明代理任意 private extension”

错。未被协议表达的 API、fd、pointer 和生命周期不能靠 generic command 重放。

### “容器无需考虑驱动 ABI”

错。容器共享 Host kernel driver，userspace CUDA/HIP driver 必须匹配。

## 16. 检查题

1. Source compatibility、binary ABI、driver UAPI 和 GPU ISA 兼容有什么区别？
2. 为什么 Vulkan ICD 可用不能证明 CUDA runtime 可用？
3. CUDA runtime 与 CUDA driver API 在兼容性判断中各代表什么？
4. 为什么 ROCm compute native context 不能简单等同于 amdgpu DRM context？
5. KFD major/minor version 为什么只是兼容性检查的开始？
6. CUDA Graph、IPC handle、VMM/P2P 为什么比单 kernel 更难代理？
7. PyTorch caching allocator 对 virtualized memory 有哪些额外要求？
8. Venus 和 GFXStream 各自保留的 API 契约是什么？
9. 为什么 capability overclaim 会在 fast path 才暴露？
10. 错误码不兼容会如何改变应用行为？
11. 容器和 VM 的 driver ABI 边界为何不同？
12. 选择 API remoting 前如何证明应用没有使用未支持私有 API？

## 17. 与其他专题的关系

- 四路径协议源码：  
  [四路径源码横向对照](../virtio-gpu/source-trace-cross-path.md)
- ROCm 社区与 ABI：  
  [Virtio-GPU Userptr 与 ROCm](../virtio-gpu/userptr-rocm-compute.md)
- KFD 源码：  
  [AMD KFD/ROCr 源码跟踪](amdkfd-source-trace.md)
- CUDA/MIG/vGPU：  
  [NVIDIA CUDA、MIG 与 vGPU](nvidia-cuda-vgpu.md)
- 调度和 fence：  
  [GPU 执行模型、队列、调度与故障隔离](gpu-execution-model.md)
- 生命周期和迁移：  
  [GPU VM 生命周期、热插拔、Checkpoint 与迁移](vm-lifecycle-migration.md)
- 容器和 Kubernetes：  
  [云原生 AI 中的 GPU 共享与虚拟化](cloud-native-gpu-sharing.md)

