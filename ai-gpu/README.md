# GPU 虚拟化到 AI 计算

本目录承接 [GPU 虚拟化文档](../README.md)，目标是把知识从“如何虚拟化图形
设备”扩展到“如何安全、可调度、可观测地共享 AI 计算设备”。

当前内容：

- [AI GPU 计算栈与虚拟化边界](ai-compute-virtualization.md)：训练/推理栈、
  显存、队列、同步、集合通信和多租户机制。
- [GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)：统一解释 Guest
  VA/GPA、Host VA、DMA/IOVA、GPU VA、VRAM/GTT、userptr/SVM/HMM、
  coherence、migration 与 teardown。
- [DMA-BUF、DMA-FENCE、Reservation 与 Sync File 源码跟踪](dma-buf-fence-source-trace.md)：
  跨 driver/device/process buffer、DMA mapping、implicit/explicit sync、
  sync_file、udmabuf 和 P2P。
- [DRM Syncobj 与 Timeline 同步源码跟踪](drm-syncobj-source-trace.md)：
  binary/timeline syncobj、fd/sync_file、eventfd、wait、transfer 和
  virtio-gpu execbuffer dependency。
- [DRM GEM 与 PRIME 源码跟踪](drm-gem-prime-source-trace.md)：driver BO、
  GEM handle、mmap offset、dma-buf fd、shmem 与 import/export 生命周期。
- [Intel Xe SVM、Userptr 与 GPU Page Fault 源码跟踪](intel-xe-svm-source-trace.md)：
  CPU address mirror VMA、mmu interval notifier、fault queue/storm、PTE/TLB
  invalidation 和 system/VRAM migration。
- [GPU 执行模型、队列、调度与故障隔离](gpu-execution-model.md)：从 command
  front end、engine、queue/doorbell、context/PASID、preemption、fence 和
  reset 解释图形与 AI workload 如何落到物理 GPU。
- [GPU 虚拟化的隔离、安全与故障域](gpu-virtualization-security.md)：覆盖恶意
  Guest、Host parser/FFI、IOMMU、VF/MIG/vGPU、数据残留、侧信道、AI 数据与
  confidential computing 的威胁模型。
- [GPU VM 生命周期、热插拔、Checkpoint 与迁移](vm-lifecycle-migration.md)：
  区分 reset、suspend/resume、snapshot、KFD CRIU、workload checkpoint 和
  live migration，并比较各虚拟化机制需要保存或重建的状态。
- [GPU 虚拟化性能分析与可观测性](gpu-performance-observability.md)：跨
  Guest、virtqueue、QEMU、renderer、Host runtime、GPU engine、HBM、display
  和 collective 建立 trace/metric/benchmark 方法与瓶颈判定树。
- [云原生 AI 中的 GPU 共享与虚拟化](cloud-native-gpu-sharing.md)：区分 VM、
  容器、device plugin/DRA/CDI、MIG、MPS、time slicing、VF 和 vGPU 的分配层
  与隔离层。
- [AI Runtime、API 与虚拟化兼容性](ai-runtime-api-compatibility.md)：区分
  source/API、binary ABI、driver UAPI 和 GPU ISA，解释 CUDA、ROCm、
  Vulkan、框架库与各虚拟化机制保留的契约。
- [OpenCL、Level Zero、SYCL 与 AI compute 虚拟化边界](opencl-levelzero-sycl-virtualization.md)：
  区分 OpenCL ICD、Intel Level Zero、SYCL backend、oneAPI 和 Vulkan compute，
  并跟踪 Rusticl/OpenCL 与 Xe UAPI。
- [GPU 虚拟化架构选型与评审 Playbook](gpu-virtualization-design-playbook.md)：
  把 workload、API、隔离、topology、lifecycle、性能和运维约束转成决策
  gate、候选架构、评分和评审模板。
- [Intel Xe GPU 虚拟化与 SR-IOV](intel-xe-sriov.md)：从 tile/GT、
  execution queue、VM_BIND/TTM、PF/VF provisioning、GuC 调度到 VF stop、
  FLR、VFIO migration 和 configfs/sysfs。
- [Intel GVT-g Mediated vGPU 源码跟踪](intel-gvt-g-source-trace.md)：mdev/KVMGT、
  shadow GTT、command parser、MMIO context、时间片调度和 reset。
- [Windows/Hyper-V GPU 虚拟化](windows-hyperv-gpu-virtualization.md)：DDA、
  GPU-P/GPU-PV、WDDM、RemoteFX 历史方案和 WSL2 GPU。
- [Vhost-user-gpu 源码跟踪](../virtio-gpu/vhost-user-gpu-source-trace.md)：
  virtio-gpu 外置后端、vhost virtqueue、display socket、dma-buf/fd 和进程隔离。
- [AMD GPU SR-IOV 与 MxGPU](amd-gpu-sriov-mxgpu.md)：PF/VF 检测、full-access
  runtime、mailbox、critical region、RLCG 间接寄存器、FLR、RAS 和 KFD queue。
- [GPU 虚拟化专家训练工作簿](expert-practice-workbook.md)：把源码追踪、
  故障树、API/ABI、性能、安全、迁移和架构选型转成可验收练习。
- [PCIe、IOMMU、ATS/PASID 与 GPU P2P](pcie-iommu-ats-p2p.md)：连接 VFIO
  group、IOMMUFD IOAS/HWPT、nested translation、GPU PASID、PRI、P2P/ACS
  和 virtio/native GPU 地址路径。
- [PCIe ATS、PASID、PRI 与 IOMMU SVA 源码跟踪](ats-pasid-pri-source-trace.md)：
  PCI capability、mm/PASID bind、IOMMU domain attach、IOPF 和 unbind/flush。
- [VFIO、IOMMUFD、SR-IOV 与 mdev 源码跟踪](vfio-iommufd-sriov-source-trace.md)：
  从 Linux 源码跟踪 legacy group/container、device cdev、IOAS、nested
  HWPT、VFIO PCI、VF enable 和 mdev 生命周期。
- [QEMU VFIO 源码跟踪](qemu-vfio-source-trace.md)：跟踪 Host fd 到 Guest
  PCI 设备、MemoryListener、BAR mmap/trap、MSI-X/IRQFD、reset 和 migration。
- [NVIDIA Grace GPU VFIO Variant 源码跟踪](nvidia-grace-vfio-source-trace.md)：
  CPU-coherent GPU memory、fake 64-bit BAR、C2C/CXL ready、reset 和 P2P。
- [GPU 显示、Compositor、WSI 与 VDI](display-wsi-virtualization.md)：从
  DRM/KMS、scanout/cursor、dmabuf、Wayland/X11/Android WSI 到 VFIO/vGPU/VDI。
- [Confidential GPU 与机密 AI 虚拟化](confidential-gpu-virtualization.md)：
  从 CPU private/shared memory、DMA/SWIOTLB、GPU private HBM、command、
  attestation、checkpoint/migration 和 collective 审查机密 AI 边界。
- [GPU 虚拟化故障诊断 Playbook](gpu-virtualization-diagnostics.md)：按设备发现、
  submit、fence、数据错误、黑屏、page fault、reset、API、性能和迁移症状
  定位最早失败层。
- [AI GPU 互连与虚拟化 Fabric](ai-interconnect-fabric-virtualization.md)：
  PCIe P2P、NVLink/NVSwitch、XGMI、UALink/NPA、RDMA/GDR、CXL 与
  VM/VF/MIG/vGPU topology。
- [GPU RAS、ECC、Reset 与 AI Job 恢复](gpu-ras-ecc-reset-recovery.md)：
  corrected/uncorrected ECC、poison、bad pages、GPU/IOMMU fault、AER/FLR、
  VF/MIG reset scope 和训练/推理恢复层级。
- [源码阅读与专家成长路线](learning-roadmap.md)：从当前 virtio-gpu 源码
  基础扩展到 ROCm/CUDA、VFIO、SR-IOV、MIG、vGPU 和 AI 集合通信。
- [AMD KFD/ROCr 计算路径源码跟踪](amdkfd-source-trace.md)：process/PASID、
  queue、doorbell、event、VRAM/GTT/userptr 和 SVM。
- [NVIDIA CUDA、MIG 与 vGPU 架构边界](nvidia-cuda-vgpu.md)：CUDA
  内存/stream/NCCL，以及 VFIO、vGPU、SR-IOV、MIG、MPS 的证据边界。
- [NCCL/RCCL Collective 源码跟踪](nccl-rccl-source-trace.md)：topology
  discovery、P2P/SHM/NET、ring/tree 和多 VM collective 边界。

本轮仍不运行 VM、构建、benchmark 或故障实验。

## 核心判断

AI 场景中，“GPU 虚拟化”至少包含四个不同问题：

1. **设备如何暴露**：PCI function、VF、mdev、virtio 设备或厂商虚拟设备。
2. **计算命令如何提交**：CUDA/HIP kernel、Vulkan compute、AQL queue 或
   driver UAPI proxy。
3. **内存如何共享**：VRAM/HBM、pinned host memory、UVA、SVM/HMM、userptr。
4. **多 GPU 通信如何进行**：PCIe/NVLink/RDMA collectives 和 topology。

任何方案只有同时说明这四层，才能判断是否适合 AI 训练、微调或推理。

## 与图形路径的关系

- VirGL/Venus/GFXStream 展示 API remoting 和资源/fence 基础。
- ROCm native context 展示 compute runtime 与 KFD/HSA 的虚拟化方向。
- VFIO/SR-IOV/MIG/vGPU 展示从整设备、PCIe function 到硬件 partition 的
  多租方式。
- AI 框架和集合通信库位于这些机制之上，不应与设备虚拟化机制混为一层。

## 推荐入口

第一次：

1. [AI GPU 计算栈与虚拟化边界](ai-compute-virtualization.md)
2. [GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)
3. [DMA-BUF、DMA-FENCE、Reservation 与 Sync File 源码跟踪](dma-buf-fence-source-trace.md)
4. [DRM Syncobj 与 Timeline 同步源码跟踪](drm-syncobj-source-trace.md)
5. [DRM GEM 与 PRIME 源码跟踪](drm-gem-prime-source-trace.md)
6. [Intel Xe SVM、Userptr 与 GPU Page Fault 源码跟踪](intel-xe-svm-source-trace.md)
7. [GPU 执行模型、队列、调度与故障隔离](gpu-execution-model.md)
8. [GPU 虚拟化的隔离、安全与故障域](gpu-virtualization-security.md)
9. [GPU VM 生命周期、热插拔、Checkpoint 与迁移](vm-lifecycle-migration.md)
10. [GPU 虚拟化性能分析与可观测性](gpu-performance-observability.md)
11. [云原生 AI 中的 GPU 共享与虚拟化](cloud-native-gpu-sharing.md)
12. [AI Runtime、API 与虚拟化兼容性](ai-runtime-api-compatibility.md)
13. [OpenCL、Level Zero、SYCL 与 AI compute 虚拟化边界](opencl-levelzero-sycl-virtualization.md)
14. [GPU 虚拟化架构选型与评审 Playbook](gpu-virtualization-design-playbook.md)
15. [Intel Xe GPU 虚拟化与 SR-IOV](intel-xe-sriov.md)
16. [Intel GVT-g Mediated vGPU 源码跟踪](intel-gvt-g-source-trace.md)
17. [Windows/Hyper-V GPU 虚拟化](windows-hyperv-gpu-virtualization.md)
18. [AMD GPU SR-IOV 与 MxGPU](amd-gpu-sriov-mxgpu.md)
19. [GPU 虚拟化专家训练工作簿](expert-practice-workbook.md)
20. [PCIe、IOMMU、ATS/PASID 与 GPU P2P](pcie-iommu-ats-p2p.md)
21. [PCIe ATS、PASID、PRI 与 IOMMU SVA 源码跟踪](ats-pasid-pri-source-trace.md)
22. [VFIO、IOMMUFD、SR-IOV 与 mdev 源码跟踪](vfio-iommufd-sriov-source-trace.md)
23. [QEMU VFIO 源码跟踪](qemu-vfio-source-trace.md)
24. [NVIDIA Grace GPU VFIO Variant 源码跟踪](nvidia-grace-vfio-source-trace.md)
25. [GPU 显示、Compositor、WSI 与 VDI](display-wsi-virtualization.md)
26. [Confidential GPU 与机密 AI 虚拟化](confidential-gpu-virtualization.md)
27. [GPU 虚拟化故障诊断 Playbook](gpu-virtualization-diagnostics.md)
28. [AI GPU 互连与虚拟化 Fabric](ai-interconnect-fabric-virtualization.md)
29. [GPU RAS、ECC、Reset 与 AI Job 恢复](gpu-ras-ecc-reset-recovery.md)
30. [Virtio-GPU Userptr 与 ROCm compute](../virtio-gpu/userptr-rocm-compute.md)
31. [ROCm compute 源码跟踪](../virtio-gpu/rocm-source-trace.md)
32. [平台方案比较](../virtio-gpu/platform-comparison.md)
33. [NVIDIA CUDA、MIG 与 vGPU 架构边界](nvidia-cuda-vgpu.md)
34. [NCCL/RCCL Collective 源码跟踪](nccl-rccl-source-trace.md)

第二次：

1. [源码阅读与专家成长路线](learning-roadmap.md)
2. [GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)
3. [GPU 执行模型、队列、调度与故障隔离](gpu-execution-model.md)
4. [AMD KFD/ROCr 源码跟踪](amdkfd-source-trace.md)
5. Linux `drivers/gpu/drm/amd/amdkfd/`
6. Linux `Documentation/gpu/drm-compute.rst`
7. Linux `Documentation/gpu/rfc/gpusvm.rst`
8. Linux `Documentation/mm/hmm.rst`
