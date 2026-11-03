# NVIDIA CUDA、MIG 与 vGPU 架构边界

## 1. 证据边界

NVIDIA CUDA userspace stack、GPU firmware 和 vGPU product internals 通常不是
完整公开源码。本文基于：

- CUDA Programming Guide；
- NVIDIA vGPU User Guide；
- MIG User Guide；
- Linux VFIO/SR-IOV/mdev kernel interfaces；
- 公开的驱动/UAPI 行为和运行时错误。

因此本文可以说明架构边界和可验证行为，但不能声称知道每一层私有
implementation。分析 NVIDIA 方案时，必须把“公开 ABI 证据”和“厂商架构描述”
分开，不能把推断写成源码事实。

## 2. CUDA 软件栈

CUDA runtime、driver API、binary ABI 和虚拟化兼容矩阵见
[AI Runtime、API 与虚拟化兼容性](ai-runtime-api-compatibility.md)。

Grace/Blackwell CPU-coherent GPU memory 和 VFIO fake BAR 的实现案例见
[NVIDIA Grace GPU VFIO Variant 源码跟踪](nvidia-grace-vfio-source-trace.md)。

概念路径：

```text
PyTorch/JAX/TensorFlow/custom CUDA
  -> CUDA runtime and vendor libraries
  -> CUDA driver API / libcuda
  -> NVIDIA kernel-mode driver/GSP
  -> GPU command processor and engines
  -> HBM and NVLink/PCIe
```

AI framework 通常不直接生成 GPU ISA。它经过：

- framework dispatcher/operator；
- kernel library，如 GEMM/convolution/attention/communication；
- compiler/runtime，如 CUDA runtime、CUDA graphs；
- CUDA driver ABI；
- KMD 和 GPU scheduler。

## 3. CUDA 主要对象

| 对象 | 作用 | 虚拟化关注点 |
| --- | --- | --- |
| Device | 物理或虚拟 GPU views | Guest 看到什么 UUID/topology |
| Context | 进程/线程 GPU 状态 | Guest context 与 Host context 隔离 |
| Module/Function | JIT/cubin 和 kernel | 编译缓存、driver version |
| Stream | 有序异步工作队列 | 低延迟 submit、跨租户调度 |
| Event | stream 完成标记 | 虚拟化 fence 映射 |
| Memory | device/pinned/managed | HBM quota、copy、fault |
| CUDA Graph | 捕获并重放工作图 | 状态迁移、句柄和调度 |
| IPC handle | 进程/设备间共享 | 多租户安全、跨 VM 不天然可用 |

## 4. CUDA 内存模型

### 4.1 主要类型

- device memory：主要位于 GPU HBM；
- pinned host memory：CPU 页锁定，可 DMA；
- mapped host memory：GPU 通过 PCIe/NVLink 访问 system memory；
- Unified Memory：同一地址由 CPU/GPU 共享，可能发生 page migration；
- stream-ordered allocation：按 stream 顺序分配/释放；
- memory pool：减少反复分配成本，AI framework caching allocator 常用。

### 4.2 AI 影响

模型权重、activation、gradient、optimizer state 和 KV cache 长期占用 HBM。
如果底层使用 Unified Memory，driver 和 GPU page fault 会引入迁移。

虚拟化必须明确：

- Guest memory 是否 pin；
- Host GPU 使用什么 DMA/sg；
- Guest/Host CPU cache 与 GPU PTE coherence；
- page fault 由 Guest KMD、Host renderer 还是 Host driver 处理；
- Host reset 时 page table 如何失效和重建。

这不只是显存容量问题。

## 5. Stream 与同步

CUDA stream 表达顺序工作；不同 stream 可以并行。AI 训练通常在多个 stream
中重叠 copy 和 compute，推理则可能通过 graph 降低 launch overhead。

虚拟化层次：

```text
CUDA API ordering
  -> Guest runtime command
  -> Guest/Host queue submit
  -> Host GPU queue
  -> events/fences
```

如果每次 submit 都经过 VM exit、QEMU、进程 IPC 和 Host runtime，deferred
decode 这类小 kernel workload 会受到很大影响。GPU-bound 大 GEMM 可能掩盖
这些成本。

## 6. Multi-GPU 与 NCCL

多 GPU AI 的路径不仅是两个独立 device：

```text
GPU A <-> NVLink/PCIe <-> GPU B
  \                         /
   NCCL/RCCL collective
```

NCCL 根据 GPU topology、P2P 能力和网络拓扑建立 communicator。虚拟化可能改变：

- Guest 看见的 PCI topology；
- P2P 是否被 IOMMU/ACS 禁用；
- VF/MIG instance 是否可互相通信；
- vGPU profile 的 NVLink 支持；
- 网络 collective 是否经过虚拟交换机；
- CUDA IPC handle 是否跨 VM 有效（通常不能直接视为有效）。

因此单卡 benchmark 不能证明训练扩展能力。

## 7. VFIO passthrough

```text
Guest NVIDIA driver
  -> Guest CUDA
  -> physical NVIDIA GPU PCI function
  -> Host IOMMU
```

特点：

- Guest 使用完整 NVIDIA 驱动；
- Guest 通常独占一张物理 GPU；
- CUDA/ROCm/私有 API 兼容性最好；
- Host 普通 graphics driver 不能同时控制该 GPU；
- reset、FLR、display、GSP firmware 和电源管理很复杂；
- migration 依赖 NVIDIA/Hypervisor/VFIO 支持，不是 VFIO 自动提供。

这是 AI 训练/单租推理的首选之一，但牺牲 GPU 密度和共享。

## 8. NVIDIA vGPU

NVIDIA vGPU 是产品/软件栈，通常包含：

- Host Virtual GPU Manager；
- Guest NVIDIA graphics/compute driver；
- Hypervisor integration；
- vGPU type/profile；
- scheduling；
- migration/管理工具；
- licence/support matrix。

Guest 看到的不是普通 virtio-gpu `1af4:1050`，而是厂商虚拟设备。

不同 GPU/世代/平台的底层实现可能不同，可能涉及：

- mediated pass-through；
- vendor-specific VFIO framework；
- SR-IOV；
- 私有 MPT 控制面。

不能从产品名推断一定使用 mdev、SR-IOV 或 MIG。必须查当前 release notes、
GPU profile 和 hypervisor support matrix。

## 9. NVIDIA MIG

MIG 把支持的数据中心 GPU 划成 GPU instance 和 compute instance/profile。
官方文档明确指出：

- 仅启 MIG mode 不足以运行 CUDA；
- 必须创建 GPU instance 和 compute instance；
- profile 决定 memory、SM、decoder/encoder 等资源；
- placement 受硬件 fragment 和 profile 顺序影响；
- MIG 配置本身不自动等于 VM assignment。

CUDA workload 可以用 MIG device UUID 选择 partition，但 VM 需要额外把
partition 暴露给 Guest；常见方式依平台可能包括 passthrough/VFIO 或厂商
manager。

## 10. SR-IOV

对于支持 GPU SR-IOV 的产品：

```text
Physical GPU PF
  -> VF
  -> VFIO/vendor assignment
  -> Guest NVIDIA driver
```

Guest 使用厂商 VF driver，Host PF 保留管理和调度职责。VF 的 VRAM、SM、
copy engine、graphics/compute 能力取决于产品 profile 和 firmware。

与 MIG 的区别：

- SR-IOV 是 PCIe function 暴露机制；
- MIG 是 GPU 内部资源 partition；
- 产品可以组合二者，但二者不是同义词。

## 11. MPS 与 time slicing

容器/Kubernetes 分配层与 MIG、MPS、time slicing 隔离层的完整比较见
[云原生 AI 中的 GPU 共享与虚拟化](cloud-native-gpu-sharing.md)。

NVIDIA MPS（Multi-Process Service）和 time slicing 主要面向 Host 上多个进程
共享 GPU：

- MPS 可减少 context switching 并提供并发执行；
- time slicing 按时间片轮转；
- 二者通常不是 PCIe VF/MIG 级别硬件隔离；
- 多租户中要考虑 fault、memory、timer/cache 和 side-channel 风险。

在 Kubernetes 中，GPU sharing label 可能代表 time slicing、MPS 或 MIG，
不能仅凭名称推断隔离等级。

## 12. 方案与 AI workload

| 方案 | 单租训练 | 多租训练 | 推理 | 隔离 | 主要限制 |
| --- | --- | --- | --- | --- | --- |
| VFIO whole GPU | 强 | 低密度 | 强 | 一 VM 独占 | 密度/迁移/reset |
| NVIDIA vGPU | 依 profile | 强 | 强 | 厂商实现 | 许可/support |
| SR-IOV VF | 依 profile | 强 | 强 | VF/硬件调度 | 支持矩阵 |
| MIG | 分区内 | 强 | 强 | 资源 partition | profile/VM 接入 |
| MPS/time slice | 弱/中 | Host 进程级 | 中 | 弱 | 缺少强 VM 隔离 |
| virtio-gpu API remoting | 依赖 API | 研究/特定 | 依赖栈 | renderer | CUDA/ROCm 兼容 |

表中的“强/弱”是架构倾向，不是未经测量的性能结论。

## 13. AI 评测矩阵

### 单 GPU

- vector add、GEMM、attention kernel；
- small-kernel launch loop；
- HBM copy/read/write；
- CUDA graph replay；
- memory allocation/fragmentation；
- model load and first token。

### 推理

- prefill 与 decode 分开；
- tokens/s、TTFT、P95/P99；
- context length sweep；
- KV cache 增长；
- 多租并发。

### 训练

- step time；
- all-reduce bandwidth/latency；
- topology/P2P；
- checkpoint 和恢复；
- failure/reset 注入。

### 方案对照

- bare metal；
- VFIO；
- vGPU；
- SR-IOV；
- MIG；
- API remoting（若 runtime 兼容）。

## 14. 可观测性

关注：

- GPU utilization、SM occupancy、HBM bandwidth；
- Xid、ECC、thermal/power；
- PCIe/NVLink bandwidth；
- CUDA context/stream/event 错误；
- vGPU scheduler/profile；
- MIG instance profile；
- NCCL topology 和 collective latency；
- Host/QEMU/renderer 边界延迟。

公开文档无法证明内部实现时，trace 和可观测接口比猜测架构更可靠。

## 15. 常见错误结论

1. Guest 的 `nvidia-smi` 显示 GPU 名称就是 VFIO。
2. vGPU 一定使用 mdev。
3. vGPU 一定使用 SR-IOV。
4. MIG 创建成功就能直接给 VM 使用。
5. MPS 能隔离不安全租户。
6. CUDA workload 能在 MIG/vGPU 上运行，就说明所有 collective/P2P 可用。
7. 单卡 GEMM 接近原生，就说明推理 decode 和训练 step 接近原生。

## 16. 后续源码/证据路线

源码可读性不同：

- Linux VFIO/SR-IOV/mdev：可读 kernel source；
- Linux NVIDIA open kernel modules：需固定具体驱动分支；
- CUDA userspace/vGPU Manager：通常依赖公开 ABI/文档；
- MIG/vGPU 特定行为：以 release notes、support matrix 和运行 trace 为准。

下一步应把本文件与：

- [平台方案比较](../virtio-gpu/platform-comparison.md)
- [AI 计算栈](ai-compute-virtualization.md)
- [学习路线](learning-roadmap.md)

一起阅读，并为每个具体 GPU 建立 fixed-version evidence sheet。
