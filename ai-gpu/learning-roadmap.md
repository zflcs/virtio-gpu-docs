# GPU 虚拟化到 AI 计算：源码与能力路线

## 1. 目标定义

目标不是替代 AI 框架工程师，而是成为能够回答以下问题的 GPU 虚拟化专家：

1. AI job 在 Guest/Host 间经过哪些 runtime、driver、queue 和内存边界？
2. virtio-gpu、native context、VFIO、SR-IOV、MIG、vGPU 各能提供什么？
3. 为什么 API remoting 对 decode/collective 可能不适用？
4. 如何在兼容性、隔离、密度、延迟和迁移之间做证据化取舍？
5. 一个方案如何被源码、ABI、failure path 和实验共同证明？

完成标准是可独立写架构评审、读关键源码、设计可复现实验并解释故障。

## 2. 当前基础

已有文档基础：

- [四条路径宏观与源码文档](../README.md)
- [四路径共用 Guest KMD 源码跟踪](../virtio-gpu/shared-kmd-source-trace.md)
- [userptr/ROCm compute](../virtio-gpu/userptr-rocm-compute.md)
- [平台虚拟化机制比较](../virtio-gpu/platform-comparison.md)
- [AI 计算栈与虚拟化边界](ai-compute-virtualization.md)

已有实验证据只覆盖 VirGL、Venus、GFXStream 的部分图形路径。它们是理解
message transport、resource、fence 和 WSI 的基础，但不能直接证明 AI
compute 虚拟化能力。

## 3. 分阶段路线

### 阶段 A：统一边界模型

目标：

- 区分 AI framework、runtime、UMD、KMD、GPU hardware；
- 能画出 PyTorch inference 的一次 kernel launch；
- 能解释 Host renderer 和设备直通的差别。

理论产出：

```text
framework -> operator/kernel -> runtime -> queue -> KMD -> GPU
```

执行层细节见
[GPU 执行模型、队列、调度与故障隔离](gpu-execution-model.md)。
API/ABI 兼容层次见
[AI Runtime、API 与虚拟化兼容性](ai-runtime-api-compatibility.md)。

验收：

- 标出 CUDA 和 ROCm 路径的对应层；
- 说明哪里会经过 virtio-gpu；
- 说明哪些对象不适合图形 API remoting。

### 阶段 B：AMD KFD/ROCr 源码

目标：

- 理解 `/dev/kfd`、process、queue、doorbell、event、VRAM/GTT/userptr；
- 能把 HSAKMT API 映射到 KFD ioctl/对象。

源码：

- [AMD KFD/ROCr 计算路径源码跟踪](amdkfd-source-trace.md)
- `linux/include/uapi/linux/kfd_ioctl.h`
- `linux/drivers/gpu/drm/amd/amdkfd/kfd_chardev.c`
- `kfd_process.c`
- `kfd_queue.c`
- `kfd_doorbell.c`
- `kfd_events.c`
- `kfd_svm.c`

验收：

- 画出一个 AQL kernel launch 的 queue/doorbell/completion；
- 解释 `KFD_IOC_ALLOC_MEM_FLAGS_USERPTR`、coherent/uncached 的含义；
- 解释 SVM page fault 和 migration 何时发生。

### 阶段 C：GPU VM 与内存

目标：

- 理解 GPU VA、PTE、HMM、SVM、userptr、DMA mapping；
- 区分映射、pin、迁移、cache coherence 和 page fault。

资料：

- [GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)
- [DMA-BUF、DMA-FENCE、Reservation 与 Sync File 源码跟踪](dma-buf-fence-source-trace.md)
- [DRM Syncobj 与 Timeline 同步源码跟踪](drm-syncobj-source-trace.md)
- [DRM GEM 与 PRIME 源码跟踪](drm-gem-prime-source-trace.md)
- [Intel Xe SVM、Userptr 与 GPU Page Fault 源码跟踪](intel-xe-svm-source-trace.md)
- [GPU 执行模型、队列、调度与故障隔离](gpu-execution-model.md)
- `linux/Documentation/mm/hmm.rst`
- `linux/Documentation/gpu/drm-compute.rst`
- `linux/Documentation/gpu/rfc/gpusvm.rst`
- `linux/Documentation/gpu/drm-vm-bind-locking.rst`

验收：

- 能区分 allocation、mapping、residency、coherence 和 lifetime；
- 能说明为什么 `dma_map_sgtable()` 成功不等于 CPU/GPU coherent；
- 能列出 userptr reuse、unmap、process exit 的释放顺序；
- 能解释 SVM 在虚拟化中为何比显存分配更复杂。
- 能区分 dma-buf object、attachment、DMA mapping、dma_fence 和 sync_file。
- 能区分 binary syncobj、timeline point、syncobj fd 和 sync_file。
- 能区分 GEM handle、mmap offset、resource ID 和 dma-buf fd。
- 能对比 AMD KFD SVM 与 Intel Xe `drm_gpusvm` 的 fault/migration 路径。

### 阶段 D：virtio-gpu compute remoting

目标：

- 理解 userptr、ROCm capset、HSAKMT proxy；
- 区分 API、UAPI、resource、queue、event、fence；
- 能审查全套 UAPI 和错误路径。

资料：

- [Vhost-user-gpu 外置后端源码跟踪](../virtio-gpu/vhost-user-gpu-source-trace.md)
- [Userptr/ROCm compute](../virtio-gpu/userptr-rocm-compute.md)
- [ROCm compute 源码跟踪](../virtio-gpu/rocm-source-trace.md)
- Linux userptr RFC v9
- QEMU userptr/ROCm RFC v4
- virglrenderer ROCm MR `!1568`

验收：

- 判断哪些版本/ABI 已合入；
- 画出 Host libhsakmt/KFD 数据路径；
- 指出 capset、feature、DMA coherence、cleanup 和 map_info 风险。
- 能区分 QEMU 进程内 renderer 与 vhost-user 外置 renderer。

### 阶段 E：硬件共享机制

目标：

- 区分 VFIO、SR-IOV、mdev、MPT、MIG、vGPU；
- 能按 workload 选型；
- 能判断 PCIe/IOMMU/GPU partition 的边界。

资料：

- [平台方案比较](../virtio-gpu/platform-comparison.md)
- [PCIe、IOMMU、ATS/PASID 与 GPU P2P](pcie-iommu-ats-p2p.md)
- [PCIe ATS、PASID、PRI 与 IOMMU SVA 源码跟踪](ats-pasid-pri-source-trace.md)
- [VFIO、IOMMUFD、SR-IOV 与 mdev 源码跟踪](vfio-iommufd-sriov-source-trace.md)
- [QEMU VFIO 源码跟踪](qemu-vfio-source-trace.md)
- [NVIDIA Grace GPU VFIO Variant 源码跟踪](nvidia-grace-vfio-source-trace.md)
- [Intel Xe GPU 虚拟化与 SR-IOV](intel-xe-sriov.md)
- [Intel GVT-g Mediated vGPU 源码跟踪](intel-gvt-g-source-trace.md)
- [Windows/Hyper-V GPU 虚拟化](windows-hyperv-gpu-virtualization.md)
- [AMD GPU SR-IOV 与 MxGPU](amd-gpu-sriov-mxgpu.md)
- `linux/Documentation/PCI/pci-iov-howto.rst`
- `linux/Documentation/driver-api/vfio*.rst`
- Intel Xe `sriov_admin` ABI
- NVIDIA MIG/vGPU 官方文档

验收：

- 解释 VF 的 PCI identity 与 mdev 的差异；
- 能区分 VFIO group/container、device cdev/IOMMUFD 和 mdev 的生命周期；
- 能从 QEMU MemoryListener 追到 Host DMA mapping；
- 能解释 CPU-coherent GPU memory、fake BAR 和 reset readiness gate；
- 能对比 GVT-g 软件 shadow/time-slice 与 Xe hardware VF；
- 能区分 PCIe PASID、IOMMU PASID domain、GPU VA 和 process mm；
- 能区分 DDA、GPU-P、WDDM 和 WSL2 GPU-PV；
- 解释 MIG 和 vGPU 不在同一层；
- 列出 AI 场景下 profile、quota、P2P 和迁移检查项。

### 阶段 F：分布式 AI 与 topology

目标：

- 理解 NCCL/RCCL 如何探测 GPU、NVLink/PCIe 和 P2P；
- 理解虚拟化对 collective 的影响。

学习对象：

- NCCL/RCCL topology graph；
- all-reduce/all-gather/all-to-all；
- GPU Direct P2P 和 RDMA；
- PCIe ACS、IOMMU、VF/MIG peer 限制；
- 多 VM 网络路径。

NVIDIA 侧可结合
[CUDA、MIG 与 vGPU 架构边界](nvidia-cuda-vgpu.md)阅读。
通信建立和 ring/tree 源码见
[NCCL/RCCL Collective 源码跟踪](nccl-rccl-source-trace.md)。
PCIe/NVLink/XGMI/UALink/RDMA/CXL 的 fabric 边界见
[AI GPU 互连与虚拟化 Fabric](ai-interconnect-fabric-virtualization.md)。

验收：

- 能解释“单卡 benchmark 很好，训练扩展效率却差”；
- 能列出 collective 最小测试矩阵；
- 能区分 compute 瓶颈、HBM 瓶颈和 interconnect 瓶颈。

### 阶段 G：AI 运行时与 workload

目标：

- 理解 PyTorch caching allocator、CUDA graph、KV cache；
- 理解训练、微调、prefill、decode 的不同延迟/内存模式。
- 区分 CUDA/HIP、OpenCL、Level Zero、SYCL 和 Vulkan compute 的运行契约。

跨 API 的 OpenCL ICD、Level Zero、SYCL backend 和 oneAPI 边界见
[OpenCL、Level Zero、SYCL 与 AI compute 虚拟化边界](opencl-levelzero-sycl-virtualization.md)。

验收：

- 解释冷启动和 steady-state 差异；
- 解释 model weight、activation、gradient、optimizer、KV cache 的生命周期；
- 能设计不同 batch/sequence length 的测量矩阵。
- 能指出 Level Zero 与 oneAPI、SYCL 与具体 backend 的区别。
- 按 Guest/transport/QEMU/Host/GPU 层拆解延迟并控制测量噪声。

测量方法见
[GPU 虚拟化性能分析与可观测性](gpu-performance-observability.md)。

### 阶段 H：生产架构

目标：

- 评估 KVM/QEMU、containers、Kubernetes device plugin/DRA；
- 设计 quota、isolation、monitoring、fault recovery 和 upgrade。

验收：

- 区分 VM 与 container 安全边界；
- 区分 MPS/time slicing、MIG、SR-IOV、vGPU；
- 能给出高密度推理和多租训练候选架构。

安全威胁模型和故障域检查表见
[GPU 虚拟化的隔离、安全与故障域](gpu-virtualization-security.md)。
设备 reset、热插拔、checkpoint 和迁移状态机见
[GPU VM 生命周期、热插拔、Checkpoint 与迁移](vm-lifecycle-migration.md)。
云原生 device plugin/DRA/CDI、MIG/MPS/time slicing 和 GPU topology 见
[云原生 AI 中的 GPU 共享与虚拟化](cloud-native-gpu-sharing.md)。
完整选型 gate、候选蓝图和评审模板见
[GPU 虚拟化架构选型与评审 Playbook](gpu-virtualization-design-playbook.md)。
显示、Compositor 和 WSI 的独立验收面见
[GPU 显示、Compositor、WSI 与 VDI](display-wsi-virtualization.md)。

## 4. 推荐 12 周节奏

| 周 | 主题 | 产出 |
| --- | --- | --- |
| 1 | 四路径源码复习 | 统一 UMD/KMD/queue/memory/fence 图 |
| 2 | AI 栈总览 | PyTorch/ROCm/GPU 的端到端图 |
| 3-4 | KFD/ROCr | queue/doorbell/event/SVM 源码笔记 |
| 5 | GPU VM/HMM/userptr | allocation 与 fault 生命周期图 |
| 6 | ROCm native context | 全栈 ABI 与 failure path 评审 |
| 7 | VFIO/SR-IOV/mdev/MPT | 设备与 I/O 模型比较 |
| 8 | MIG/vGPU/MPS | 资源切分与调度比较 |
| 9 | 多 GPU topology | NCCL/RCCL 与 P2P 检查表 |
| 10 | 推理 workload | prefill/decode 测量设计 |
| 11 | 训练 workload | collective/checkpoint 测量设计 |
| 12 | Capstone | 选型、源码审查、实验计划和风险报告 |

## 5. 实验解禁后的最小实验阶梯

当前先不执行；以下仅作为未来计划。

文档证据等级、源码 commit 和复核规则见
[GPU 虚拟化文档证据登记册](../EVIDENCE_REGISTER.md)。

### E1：单 GPU 软件对照

- Host native；
- VFIO；
- API remoting/ROCm proposal（若可用）。

workload：

- vector add；
- GEMM；
- 小 kernel 串行；
- 大 tensor copy。

### E2：显存与同步

- pinned vs pageable；
- userptr vs device allocation；
- map/unmap 频率；
- fence round-trip。

### E3：推理

- 固定模型和 batch；
- prefill/decode 分开；
- tokens/s 和 P95/P99；
- KV cache 增长。

### E4：训练/集合通信

- all-reduce size sweep；
- NVLink/PCIe/虚拟机网络；
- 多租户干扰；
- checkpoint 与恢复。

### E5：隔离与故障

- 单租户 OOM/device lost；
- reset 对其他 VM 的影响；
- queue/window teardown；
- migration/snapshot 失败恢复。

## 6. 专家验收

配套源码追踪、故障树和架构设计练习见
[GPU 虚拟化专家训练工作簿](expert-practice-workbook.md)。

能完成以下四项才算阶段性达标：

1. **源码**：从 Guest framework 到 Host GPU 画出并解释完整 call chain。
2. **架构**：为三类 AI workload 选择虚拟化机制并说明取舍。
3. **故障**：从任意一层症状定位 queue、memory、fence 或 topology。
4. **证据**：设计实验区分 GPU-bound、HBM-bound、launch-bound 和
   collective-bound。

症状驱动的排障顺序见
[GPU 虚拟化故障诊断 Playbook](gpu-virtualization-diagnostics.md)。

## 7. 持续维护

每篇更新记录：

- 组件版本/commit；
- UAPI/feature/capset；
- 合入状态；
- 硬件/profile/driver 支持矩阵；
- 已知 limitation 和失败路径；
- 是否有独立证据。

目标不是追求更多术语，而是能对任何新 GPU 虚拟化/AI 方案快速问出正确问题
并验证答案。
