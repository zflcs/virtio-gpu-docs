# GPU 虚拟化文档

本目录是后续 GPU 虚拟化整理内容的统一入口。文档按“先宏观、再源码、再做
社区与方案横向比较”的顺序组织。

全库的证据等级、固定 commit、社区状态和上游复核规则见
[EVIDENCE_REGISTER.md](EVIDENCE_REGISTER.md)。

本轮没有运行 VM、构建、性能或故障复现实验。已有实验证据仍来自仓库根目录
Phase 文档和 `build/traces/`。

## 1. 文档结构

### 1.1 四条路径的宏观文档

| 路径 | 文档 | 核心回答 |
| --- | --- | --- |
| VirGL | [virgl-overview.md](virtio-gpu/virgl-overview.md) | OpenGL API remoting 如何工作 |
| Venus | [venus-overview.md](virtio-gpu/venus-overview.md) | Vulkan ring/proxy renderer 如何工作 |
| GFXStream | [gfxstream-overview.md](virtio-gpu/gfxstream-overview.md) | 多 API streaming 与 Android 图形栈 |
| Rutabaga | [rutabaga-overview.md](virtio-gpu/rutabaga-overview.md) | QEMU 侧 VGI、Rust component 和 C FFI |

这里的 `N = 4`。GFXStream 与 Rutabaga 必须分开理解：

- GFXStream 是 Host graphics API streaming backend；
- Rutabaga 是 VMM 侧的 component/VGI adapter；
- QEMU `virtio-gpu-rutabaga` 是二者的 QEMU glue。

### 1.2 逐路径源码跟踪

| 路径 | 源码文档 | 主要跟踪内容 |
| --- | --- | --- |
| VirGL | [virgl-source-trace.md](virtio-gpu/virgl-source-trace.md) | GL encode → DRM ioctl → vrend → Host GL |
| Venus | [venus-source-trace.md](virtio-gpu/venus-source-trace.md) | Vulkan → ring → proxy → vkr → Host Vulkan |
| GFXStream | [gfxstream-source-trace.md](virtio-gpu/gfxstream-source-trace.md) | ICD → ASG ring → Rutabaga → GFXStream backend |
| Rutabaga | [rutabaga-source-trace.md](virtio-gpu/rutabaga-source-trace.md) | QEMU → C FFI → Rust core → component |

四条路径在 Guest 内共用 Linux `virtio_gpu` KMD。控制队列、resource/blob、
execbuffer、virtqueue 和 fence completion 的逐函数分析统一放在
[shared-kmd-source-trace.md](virtio-gpu/shared-kmd-source-trace.md)；
每份路径文档只解释该路径对这些公共机制的差异化使用。

[source-trace-cross-path.md](virtio-gpu/source-trace-cross-path.md) 用于横向
比较四者的 UMD/KMD、控制面、数据面、内存和 fence，并附带 native DRM
context 作为扩展对照。

Native DRM context 另有独立文档，但仍是扩展路径，不改变四主路径的定义：

- [Native DRM context 宏观文档](virtio-gpu/native-drm-overview.md)
- [Native DRM context 源码跟踪](virtio-gpu/native-drm-source-trace.md)

vhost-user-gpu 是同一 virtio-gpu Guest ABI 的外置后端部署模式：

- [Vhost-user-gpu 源码跟踪](virtio-gpu/vhost-user-gpu-source-trace.md)

### 1.3 社区新 feature

[userptr-rocm-compute.md](virtio-gpu/userptr-rocm-compute.md)
和 [ROCm compute 源码跟踪](virtio-gpu/rocm-source-trace.md)

解释：

- 为什么 ROCm compute 需要 virtio-gpu userptr；
- Guest UMD/KMD、QEMU、renderer、Host libhsakmt 如何配合；
- 为什么它共享的是 Host GPU compute 能力，而不只是渲染能力；
- Linux RFC v9、QEMU RFC v4、virglrenderer `!1568` 的当前状态与 ABI 错位。

### 1.4 平台方案比较

[platform-comparison.md](virtio-gpu/platform-comparison.md)

从 Guest ABI、PCIe function、GPU 内部资源、Host 执行边界和 I/O 基础设施
比较：

- virtio-gpu；
- VFIO；
- SR-IOV；
- mdev；
- MPT（Mediated Pass-Through）；
- MIG；
- 厂商 vGPU。

### 1.5 AI 计算延伸

AI 专题的总入口是 [ai-gpu/README.md](ai-gpu/README.md)。

[AI GPU 计算栈与虚拟化边界](ai-gpu/ai-compute-virtualization.md) 和
[源码阅读与成长路线](ai-gpu/learning-roadmap.md) 把虚拟化机制映射到训练、
微调、推理、HBM/VRAM、SVM、queue/event 和多卡集合通信。

[GPU 显存、地址空间与一致性语义](ai-gpu/gpu-memory-semantics.md) 统一解释
Guest CPU VA/GPA、Host VA、DMA/IOVA、GPU VA、VRAM/GTT、userptr/SVM/HMM、
cache coherence、migration/eviction 和各虚拟化方案的显存边界。

[DMA-BUF、DMA-FENCE、Reservation 与 Sync File 源码跟踪](ai-gpu/dma-buf-fence-source-trace.md)
补充跨 driver/device/process 共享对象、DMA mapping、implicit/explicit sync、
sync_file、udmabuf、formats/modifiers 和 P2P 的共同底座。

[DRM Syncobj 与 Timeline 同步源码跟踪](ai-gpu/drm-syncobj-source-trace.md)
跟踪 binary/timeline syncobj、handle/fd/sync_file、wait/eventfd/query/transfer，
以及 virtio-gpu execbuffer input/output dependency。

[DRM GEM 与 PRIME 源码跟踪](ai-gpu/drm-gem-prime-source-trace.md) 区分 driver
BO、GEM object、GEM handle、fake mmap offset 和 dma-buf fd，并跟踪 PRIME
import/export、shmem backing 与 reservation 生命周期。

[Intel Xe SVM、Userptr 与 GPU Page Fault 源码跟踪](ai-gpu/intel-xe-svm-source-trace.md)
补足另一套现代 GPU SVM 实现，逐段分析 CPU address mirror VMA、mmu interval
notifier、fault queue/storm、PTE/TLB invalidation 和 VRAM/SMEM migration。

[GPU 执行模型、队列、调度与故障隔离](ai-gpu/gpu-execution-model.md) 统一解释
command front end、engine、queue/ring/doorbell、context/PASID、preemption、
fence、reset 和 AI 小 kernel 的提交开销。

[GPU 虚拟化的隔离、安全与故障域](ai-gpu/gpu-virtualization-security.md) 从
威胁模型分析 Guest/Host 信任边界、parser/FFI、IOMMU、VF/MIG/vGPU、数据残留、
侧信道、AI 数据和 confidential computing。

[GPU VM 生命周期、热插拔、Checkpoint 与迁移](ai-gpu/vm-lifecycle-migration.md)
区分 reset、suspend/resume、snapshot、KFD CRIU、workload checkpoint 和 live
migration，并说明各虚拟化机制需要保存、重建或必须丢弃的状态。

[GPU 虚拟化性能分析与可观测性](ai-gpu/gpu-performance-observability.md) 给出
跨 Guest、virtqueue、QEMU、renderer、Host runtime、GPU engine、HBM、display
和 collective 的计时、trace、基准与瓶颈判定方法。

[云原生 AI 中的 GPU 共享与虚拟化](ai-gpu/cloud-native-gpu-sharing.md) 区分
VM、容器和 Kubernetes 分配层，分析 device plugin/DRA/CDI、MIG、MPS、
time slicing、VF、vGPU、quota、topology 和租户安全。

[AI Runtime、API 与虚拟化兼容性](ai-gpu/ai-runtime-api-compatibility.md)
区分 source/API、binary ABI、driver UAPI 和 GPU ISA，解释 CUDA、ROCm、
Vulkan、PyTorch/custom extension 与各虚拟化机制保留的契约。

[OpenCL、Level Zero、SYCL 与 AI compute 虚拟化边界](ai-gpu/opencl-levelzero-sycl-virtualization.md)
补充跨厂商 compute API、Intel low-level runtime 和 SYCL backend 模型，
并以 Mesa Rusticl 与 Xe UAPI 对照 OpenCL kernel 的源码执行路径。

[GPU 虚拟化架构选型与评审 Playbook](ai-gpu/gpu-virtualization-design-playbook.md)
把需求分析、API/ABI、隔离、topology、lifecycle、性能和运维转换为 decision
gate、候选架构、评分、验收证据和 rollout/rollback 模板。

[Intel Xe GPU 虚拟化与 SR-IOV](ai-gpu/intel-xe-sriov.md) 从 Linux Xe KMD
源码分析 tile/GT、execution queue、VM_BIND/TTM、PF/VF provisioning、
GuC 调度、VF stop/FLR、VFIO migration 和 configfs/sysfs。

[Intel GVT-g Mediated vGPU 源码跟踪](ai-gpu/intel-gvt-g-source-trace.md) 以
i915 GVT-g 为例分析 mdev/KVMGT、GTT shadow、command parser、MMIO context、
weighted time slice 和 reset，作为软件 mediated vGPU 的历史基线。

[Windows/Hyper-V GPU 虚拟化](ai-gpu/windows-hyperv-gpu-virtualization.md)
区分 DDA、GPU-P/GPU-PV、WDDM、RemoteFX 历史方案和 WSL2 GPU，并说明它们的
公开证据边界、AI API 兼容与迁移限制。

[AMD GPU SR-IOV 与 MxGPU](ai-gpu/amd-gpu-sriov-mxgpu.md) 从 amdgpu/KFD 源码
分析 PF/VF 检测、full-access/runtime、mailbox、critical region、indirect
register、FLR、RAS 和 Guest ROCm queue/VM。

[GPU 虚拟化专家训练工作簿](ai-gpu/expert-practice-workbook.md) 把源码追踪、
故障树、API/ABI、性能、安全、迁移和架构选型整理成可自评练习。

[PCIe、IOMMU、ATS/PASID 与 GPU P2P](ai-gpu/pcie-iommu-ats-p2p.md) 解释
VFIO group、IOMMUFD IOAS/HWPT、nested translation、PASID/PRI、ACS、P2P
和 GPU 页表之间的边界。

[PCIe ATS、PASID、PRI 与 IOMMU SVA 源码跟踪](ai-gpu/ats-pasid-pri-source-trace.md)
逐函数分析 PCI ATS/PASID/PRI capability、per-process PASID、IOMMU domain
attach、IOPF fault queue、unbind/flush 和 VFIO/IOMMUFD 接入。

[VFIO、IOMMUFD、SR-IOV 与 mdev 源码跟踪](ai-gpu/vfio-iommufd-sriov-source-trace.md)
进一步跟踪 device cdev、legacy group/container、IOAS、nested HWPT、VFIO
PCI region/interrupt/reset、PCI VF 创建和 mdev parent/type/device 生命周期。

[QEMU VFIO 源码跟踪](ai-gpu/qemu-vfio-source-trace.md) 衔接 Linux kernel
侧，解释 QEMU 如何实现 Guest AddressSpace MemoryListener、BAR/mmio 快慢路径、
MSI-X/IRQFD、hot reset 和 VFIO migration stream。

[NVIDIA Grace GPU VFIO Variant 源码跟踪](ai-gpu/nvidia-grace-vfio-source-trace.md)
以 GH200/GB200/GB300 的 VFIO driver 为例，分析 CPU-coherent GPU memory、
fake 64 位 BAR、C2C/CXL readiness、reset 后首访问和 dma-buf/P2P。

[GPU 显示、Compositor、WSI 与 VDI](ai-gpu/display-wsi-virtualization.md)
解释 DRM/KMS、scanout/cursor、damage、dma-buf、Wayland/X11/Android WSI、
VFIO/vGPU display 和 VDI 编码路径。

[Confidential GPU 与机密 AI 虚拟化](ai-gpu/confidential-gpu-virtualization.md)
区分 CPU TEE、GPU private memory/command、DMA/SWIOTLB、attestation、
checkpoint/migration、collective 和 telemetry 的端到端信任边界。

[GPU 虚拟化故障诊断 Playbook](ai-gpu/gpu-virtualization-diagnostics.md)
从设备发现、context、submit、fence、data corruption、display、fault/reset、
API、性能、collective 和 migration 症状定位最早失败边界。

[AI GPU 互连与虚拟化 Fabric](ai-gpu/ai-interconnect-fabric-virtualization.md)
比较 PCIe P2P、NVLink/NVSwitch、XGMI、UALink/NPA、RDMA/GDR、CXL 以及
VM/VF/MIG/vGPU topology。

[GPU RAS、ECC、Reset 与 AI Job 恢复](ai-gpu/gpu-ras-ecc-reset-recovery.md)
区分 corrected/uncorrected ECC、poison、bad pages、GPU/IOMMU fault、
AER/FLR、VF/MIG 故障域和 workload/collective recovery。

### 1.6 证据与维护

[GPU 虚拟化文档证据登记册](EVIDENCE_REGISTER.md) 记录：

- 固定源码 commit；
- E0-E5 证据等级；
- implemented/merged/opened/proposed/released 状态；
- userptr/ROCm/QEMU/virglrenderer 的当前状态与开放问题；
- 历史实验和本轮只读文档的边界；
- 上游变化后需要复核的源码和结论。

KFD 的具体源码路径见
[AMD KFD/ROCr 计算路径源码跟踪](ai-gpu/amdkfd-source-trace.md)。
NVIDIA CUDA/vGPU/MIG 的公开证据边界见
[NVIDIA CUDA、MIG 与 vGPU 架构](ai-gpu/nvidia-cuda-vgpu.md)。
collective/topology 源码见
[NCCL/RCCL Collective 源码跟踪](ai-gpu/nccl-rccl-source-trace.md)。

## 2. 推荐阅读顺序

第一次阅读，先建立每条路径的共同框架：

1. [VirGL 宏观文档](virtio-gpu/virgl-overview.md)
2. [Venus 宏观文档](virtio-gpu/venus-overview.md)
3. [GFXStream 宏观文档](virtio-gpu/gfxstream-overview.md)
4. [Rutabaga 宏观文档](virtio-gpu/rutabaga-overview.md)

第二次阅读，按路径分别对照函数和数据结构：

5. [VirGL 源码跟踪](virtio-gpu/virgl-source-trace.md)
6. [Venus 源码跟踪](virtio-gpu/venus-source-trace.md)
7. [GFXStream 源码跟踪](virtio-gpu/gfxstream-source-trace.md)
8. [Rutabaga 源码跟踪](virtio-gpu/rutabaga-source-trace.md)

四份路径文档都依赖同一个 Guest KMD；在横向比较前阅读
[共用 Linux KMD 源码跟踪](virtio-gpu/shared-kmd-source-trace.md)，可以把
“路径特有逻辑”和“virtio-gpu transport 公共逻辑”明确分开。

9. [四路径源码横向对照](virtio-gpu/source-trace-cross-path.md)
10. [Native DRM context 宏观文档](virtio-gpu/native-drm-overview.md)
11. [Native DRM context 源码跟踪](virtio-gpu/native-drm-source-trace.md)
12. [Vhost-user-gpu 源码跟踪](virtio-gpu/vhost-user-gpu-source-trace.md)

第三次阅读，扩展到计算和硬件方案：

13. [Userptr/ROCm compute](virtio-gpu/userptr-rocm-compute.md)
14. [平台方案比较](virtio-gpu/platform-comparison.md)
15. [ROCm compute 源码跟踪](virtio-gpu/rocm-source-trace.md)

第四次阅读，向 AI compute 扩展：

16. [AI GPU 计算栈与虚拟化边界](ai-gpu/ai-compute-virtualization.md)
17. [GPU 显存、地址空间与一致性语义](ai-gpu/gpu-memory-semantics.md)
18. [DMA-BUF、DMA-FENCE、Reservation 与 Sync File 源码跟踪](ai-gpu/dma-buf-fence-source-trace.md)
19. [DRM Syncobj 与 Timeline 同步源码跟踪](ai-gpu/drm-syncobj-source-trace.md)
20. [DRM GEM 与 PRIME 源码跟踪](ai-gpu/drm-gem-prime-source-trace.md)
21. [Intel Xe SVM、Userptr 与 GPU Page Fault 源码跟踪](ai-gpu/intel-xe-svm-source-trace.md)
22. [GPU 执行模型、队列、调度与故障隔离](ai-gpu/gpu-execution-model.md)
23. [GPU 虚拟化的隔离、安全与故障域](ai-gpu/gpu-virtualization-security.md)
24. [GPU VM 生命周期、热插拔、Checkpoint 与迁移](ai-gpu/vm-lifecycle-migration.md)
25. [GPU 虚拟化性能分析与可观测性](ai-gpu/gpu-performance-observability.md)
26. [云原生 AI 中的 GPU 共享与虚拟化](ai-gpu/cloud-native-gpu-sharing.md)
27. [AI Runtime、API 与虚拟化兼容性](ai-gpu/ai-runtime-api-compatibility.md)
28. [OpenCL、Level Zero、SYCL 与 AI compute 虚拟化边界](ai-gpu/opencl-levelzero-sycl-virtualization.md)
29. [GPU 虚拟化架构选型与评审 Playbook](ai-gpu/gpu-virtualization-design-playbook.md)
30. [Intel Xe GPU 虚拟化与 SR-IOV](ai-gpu/intel-xe-sriov.md)
31. [Intel GVT-g Mediated vGPU 源码跟踪](ai-gpu/intel-gvt-g-source-trace.md)
32. [Windows/Hyper-V GPU 虚拟化](ai-gpu/windows-hyperv-gpu-virtualization.md)
33. [AMD GPU SR-IOV 与 MxGPU](ai-gpu/amd-gpu-sriov-mxgpu.md)
34. [GPU 虚拟化专家训练工作簿](ai-gpu/expert-practice-workbook.md)
35. [PCIe、IOMMU、ATS/PASID 与 GPU P2P](ai-gpu/pcie-iommu-ats-p2p.md)
36. [PCIe ATS、PASID、PRI 与 IOMMU SVA 源码跟踪](ai-gpu/ats-pasid-pri-source-trace.md)
37. [VFIO、IOMMUFD、SR-IOV 与 mdev 源码跟踪](ai-gpu/vfio-iommufd-sriov-source-trace.md)
38. [QEMU VFIO 源码跟踪](ai-gpu/qemu-vfio-source-trace.md)
39. [NVIDIA Grace GPU VFIO Variant 源码跟踪](ai-gpu/nvidia-grace-vfio-source-trace.md)
40. [GPU 显示、Compositor、WSI 与 VDI](ai-gpu/display-wsi-virtualization.md)
41. [Confidential GPU 与机密 AI 虚拟化](ai-gpu/confidential-gpu-virtualization.md)
42. [GPU 虚拟化故障诊断 Playbook](ai-gpu/gpu-virtualization-diagnostics.md)
43. [AI GPU 互连与虚拟化 Fabric](ai-gpu/ai-interconnect-fabric-virtualization.md)
44. [GPU RAS、ECC、Reset 与 AI Job 恢复](ai-gpu/gpu-ras-ecc-reset-recovery.md)
45. [GPU 虚拟化到 AI 的源码与能力路线](ai-gpu/learning-roadmap.md)
46. [AMD KFD/ROCr 计算路径源码跟踪](ai-gpu/amdkfd-source-trace.md)
47. [NVIDIA CUDA、MIG 与 vGPU 架构边界](ai-gpu/nvidia-cuda-vgpu.md)
48. [NCCL/RCCL Collective 源码跟踪](ai-gpu/nccl-rccl-source-trace.md)
49. [GPU 虚拟化文档证据登记册](EVIDENCE_REGISTER.md)

## 3. 四条路径的快速区别

| 路径 | Guest UMD | QEMU/Host adapter | Host renderer |
| --- | --- | --- | --- |
| VirGL | Mesa VirGL Gallium driver | QEMU VirGL adapter | vrend -> Host OpenGL |
| Venus | Mesa virtio Vulkan ICD | virglrenderer proxy -> render server | `vkr` -> Host Vulkan |
| GFXStream | Guest GFXStream ICD/encoder | QEMU Rutabaga -> FFI | `libgfxstream_backend` |
| Rutabaga | 无独立 UMD；使用所选 component | QEMU + Rust FFI/component | 由 component 决定，当前重点是 GFXStream |

Rutabaga 是四条路径中唯一的 Host-side 框架层，不是另一套 Guest GPU 设备
协议。这个区别是理解 QEMU GFXStream 部署的关键。

## 4. 四条路径都必须回答的问题

1. Guest 应用调用什么 API？
2. Guest UMD 负责编码什么？
3. Guest KMD 是否解析图形命令？
4. 数据通过命令 payload、共享 ring、blob 还是 dma-buf 传输？
5. Host GPU 由 renderer、Vulkan/GL driver 还是 native KMD 使用？
6. fence 在哪个对象/ring/timeline 上完成？
7. 合成与 scanout 在 Guest 还是 Host？
8. 哪个对象的所有权跨 Guest/Host 边界？
9. 当前版本是否支持 reset、hibernation 和 live migration？
10. 安全边界是软件解析器、进程、IOMMU、PCIe VF 还是硬件 partition？

宏观文档负责建立答案框架，源码文档负责给出证据位置。

## 5. 已有实验证据

这些文档保留此前的运行结果，本次没有重新运行：

| 文档 | 证据 |
| --- | --- |
| [阶段 0：VirGL](../PHASE_0_HOST_QEMU_VIRTIO_GPU_PATH.md) | OpenGL 路径、QEMU trace 和 Host GPU 使用 |
| [阶段 1：Guest resource/blob](../PHASE_1_GUEST_DRM_RESOURCES.md) | Guest UAPI 和 resource 生命周期 |
| [阶段 2：QEMU 边界](../PHASE_2_QEMU_DEVICE_BOUNDARY.md) | virtqueue、内存映射、fence 和 migration |
| [阶段 3：Venus](../PHASE_3_VENUS_VULKAN_PATH.md) | Venus ring、proxy 和运行记录 |
| [阶段 4：GFXStream/Rutabaga](../PHASE_4_GFXSTREAM_RUTABAGA.md) | GFXStream 构建、离屏和 WSI |
| [KVM Venus 诊断](../KVM_VENUS_DIAGNOSIS.md) | Guest Mesa 版本和 fence A/B |
| [GFXStream WSI 诊断](../GFXSTREAM_NATIVE_WSI_XID69_DIAGNOSIS.md) | dma-buf/modifier/validation 分层问题 |
| [性能基线](../PERFORMANCE_AND_HARDWARE_BASELINE.md) | 当前硬件与历史数据 |
| [GPU 虚拟化路线](../GPU_VIRTUALIZATION_LEARNING_PATH.md) | 长期学习与实验安排 |

## 6. 社区状态摘要

截至 2026-10-29：

| 项目 | 状态 | 文档中的判断 |
| --- | --- | --- |
| Linux userptr | RFC v9，未合入 | UAPI 和错误路径仍在评审 |
| QEMU userptr/ROCm | RFC v4，ABI 落后 | 不能直接配 Linux v9 |
| virglrenderer ROCm | MR `!1568` opened | 仍需 rebase、flag 修复和 CI |
| 当前 Linux blob alignment | 已合入 | 可影响 userptr 页/地址对齐 |
| ROCm/KFD 虚拟化 | 向 DRM SVM 设计演进 | 不是单个 ioctl 能完成 |

状态会变化。后续更新时记录组件、系列版本、日期、commit 和是否 merged。

## 7. 推荐产出

读完文档后，建议为每条路径写一页自己的图：

```text
Guest application
  -> Guest UMD
  -> Guest KMD
  -> virtio mechanism
  -> QEMU/Host adapter
  -> Host renderer
  -> Host GPU
  -> completion/present path
```

每个箭头旁边标注：

- 使用的 UAPI/协议；
- 数据所有权；
- 同步对象；
- 失败会在哪里暴露；
- 对应源码函数。

这是从“知道名词”走向“能设计、调试和评审 GPU 虚拟化系统”的核心练习。
