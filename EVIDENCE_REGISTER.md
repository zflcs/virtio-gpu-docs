# GPU 虚拟化文档证据登记册

## 1. 用途

本登记册回答三个问题：

1. 文档中的结论基于哪个固定源码版本或公开资料？
2. 它是主线实现、RFC、开放 MR、厂商文档，还是历史运行结果？
3. 上游变化后，需要重新检查哪些文件和结论？

它是证据索引，不替代专题文档。

## 2. 快照

- 文档快照日期：2026-10-29
- 时区：Asia/Shanghai
- 本轮实验策略：只读源码/文档，不运行 VM、GPU、benchmark、迁移或故障注入

## 3. 固定源码版本

| 项目 | Commit |
| --- | --- |
| Linux | `6b4b543fa18757297ac756a6fe6e64f506b7844a` |
| Mesa | `ec7b2f322abbcd593d600a14afe2d2773512af4e` |
| virglrenderer | `e0a1caf3b147bb26dfe524dbb85ded1177e7e49e` |
| QEMU | `d73149283205645625bfa9fe029cfb68e4c4a9ae` |
| NCCL | `3e46c089f4ae4732722a419eb6096aead7e47a30` |
| RCCL | `57e58688f44c77076ad536ef1f6b68741fc6e694` |

版本用于界定源码结论。发行版内核、厂商 driver 或产品可能基于其他 revision。

DMA-BUF、DMA-FENCE、DMA reservation、sync_file 与 udmabuf 的共享源码证据见
[DMA-BUF、DMA-FENCE、Reservation 与 Sync File 源码跟踪](ai-gpu/dma-buf-fence-source-trace.md)。

binary/timeline syncobj 与 virtio execbuffer dependency 的源码证据见
[DRM Syncobj 与 Timeline 同步源码跟踪](ai-gpu/drm-syncobj-source-trace.md)。

GEM handle、mmap offset 和 PRIME 对象生命周期的源码证据见
[DRM GEM 与 PRIME 源码跟踪](ai-gpu/drm-gem-prime-source-trace.md)。

## 4. 证据等级

| 等级 | 定义 | 可用于 |
| --- | --- | --- |
| E0 | 博客、路线图、营销材料 | 方向线索 |
| E1 | 公开 patch/RFC/draft MR | 方案和 ABI 讨论 |
| E2 | 最新 patch/MR，包含实现和 review | 代码行为、已知缺陷 |
| E3 | 相关组件 ABI 对齐，维护者接受，CI 通过 | merge-ready 判断 |
| E4 | 合入主线并进入发布产品/发行版 | 可用性判断 |
| E5 | 目标平台上独立正确性、性能、安全和迁移验证 | 生产结论 |

不同维度应分别评级：

```text
source evidence
  != performance evidence
  != security evidence
  != migration evidence
```

## 5. 结论状态词

文档统一使用：

- **implemented**：固定源码中可读到的实现；
- **merged**：已进入相应主线；
- **released**：出现在具体发布版本；
- **proposed**：RFC/patch/MR；
- **opened**：MR 仍开放；
- **vendor-defined**：实现依厂商/产品；
- **platform-dependent**：依硬件、firmware、hypervisor 或驱动；
- **unknown**：缺少当前证据。

“文件存在”“MR opened”“能编译”都不等于 released/production-proven。

## 6. 四主路径证据

### 6.1 VirGL

固定源码：

- `mesa/src/gallium/drivers/virgl/`
- `mesa/src/gallium/winsys/virgl/drm/`
- `linux/drivers/gpu/drm/virtio/`
- `qemu/hw/display/virtio-gpu*.c`
- `virglrenderer/src/vrend/`

可证明：

- Guest GL/Gallium command encoding；
- virtio execbuffer/control queue；
- QEMU payload copy；
- Host vrend decode 和 Host GL 调用；
- Guest virtio fence 返回。

不能直接证明：

- 所有 workload 性能；
- Host GL/driver 无错误；
- VirGL live migration。

文档：

- [宏观](virtio-gpu/virgl-overview.md)
- [源码](virtio-gpu/virgl-source-trace.md)

### 6.2 Venus

固定源码：

- `mesa/src/virtio/vulkan/`
- `virglrenderer/src/proxy/`
- `virglrenderer/src/venus/`
- QEMU VirGL adapter

可证明：

- Vulkan ICD 初始化、ring layout、tail/head 内存序；
- proxy render server 通信；
- Host `vkr` dispatch；
- ring seqno 与 Vulkan fence 分层。

不能直接证明：

- 所有 Vulkan extension；
- 任意 GPU-bound workload 接近原生；
- proxy renderer state 可迁移。

文档：

- [宏观](virtio-gpu/venus-overview.md)
- [源码](virtio-gpu/venus-source-trace.md)

### 6.3 GFXStream

固定源码：

- `mesa/src/gfxstream/guest/`
- GFXStream host backend 基线 `0ab696da...`
- `rutabaga_gfx-0.1.3`
- `rutabaga_gfx_ffi-0.1.3`
- `linux/drivers/gpu/drm/virtio/virtgpu_display.c`
- `linux/drivers/gpu/drm/virtio/virtgpu_plane.c`
- QEMU `virtio-gpu-base.c` / `virtio-gpu.c` display paths

可证明：

- Guest GFXStream ICD、ResourceTracker、ASG ring；
- QEMU Rutabaga C ABI；
- Rust component 到 `stream_renderer_*`；
- fence callback 线程边界。

不能直接证明：

- 任意 GFXStream API extension；
- native WSI 在所有 compositor 上可用；
- QEMU live migration。

文档：

- [宏观](virtio-gpu/gfxstream-overview.md)
- [源码](virtio-gpu/gfxstream-source-trace.md)

### 6.4 Rutabaga

固定源码：

- `qemu/hw/display/virtio-gpu-rutabaga.c`
- `rutabaga_gfx_ffi-0.1.3/src/lib.rs`
- `rutabaga_gfx-0.1.3/src/`

可证明：

- QEMU realize/migration blocker；
- FFI object/length/error contract；
- context/resource/blob/fence handoff；
- Rust component abstraction。

不能直接证明：

- 任意 component 的功能；
- snapshot API 等于完整 VM migration。

文档：

- [宏观](virtio-gpu/rutabaga-overview.md)
- [源码](virtio-gpu/rutabaga-source-trace.md)

### 6.5 四路径横向

[source-trace-cross-path.md](virtio-gpu/source-trace-cross-path.md) 汇总：

- Guest UMD/KMD；
- control/data/completion plane；
- resource/blob 所有权；
- fence 与故障定位；
- native DRM context 扩展对照。

四主路径共用的 Guest Linux KMD 证据集中在
[shared-kmd-source-trace.md](virtio-gpu/shared-kmd-source-trace.md)。它证明
当前固定 Linux commit 中的 feature/queue/context/blob/execbuffer/fence 与
suspend/restore 行为；“各路径如何使用这些公共机制”仍由四份路径源码文档
分别证明。

## 7. Native DRM context 证据

固定源码：

- `mesa/src/virtio/vdrm/`
- `mesa/src/amd/common/virtio/`
- `virglrenderer/src/drm/drm_context.c`
- `virglrenderer/src/drm/amdgpu/`
- `virglrenderer/src/drm/{msm,i915,asahi,panfrost}/`
- QEMU `virtio-gpu-gl.c`

可证明：

- Capset 6 和 context type；
- control shmem/CCMD；
- AMDGPU shim 的 GEM/VA/CS/syncobj 转换；
- generic dispatch 边界检查与 shadow response；
- Host native context/timeline/fence；
- dma-buf import/export。

不能直接证明：

- 物理 PCI 隔离；
- 任意 Guest native driver 二进制兼容；
- CUDA/HIP 兼容；
- transparent live migration。

状态：

- 固定源码中多 backend implemented；
- Host 启用条件受 EGL、render node、virglrenderer version 约束；
- 具体硬件运行结果不在本轮范围内。

文档：

- [宏观](virtio-gpu/native-drm-overview.md)
- [源码](virtio-gpu/native-drm-source-trace.md)

## 8. Userptr/ROCm 社区状态

公开状态快照：

| 组件 | 状态 | 评级 |
| --- | --- | --- |
| Linux userptr | RFC v9，2026-09-24，未合入 | E2 |
| Linux KFD/DRM SVM 方向 | 设计/评审中 | E1 |
| QEMU userptr/ROCm | RFC v4，2026-01-07，ABI 落后 | E1/E2 |
| virglrenderer ROCm MR `!1568` | opened，需 rebase | E2 |
| ROCm runtime virtio 前端 | 部分组件在 ROCm 仓库推进 | 未达 E3 |
| 生产可用性 | 未达到全栈 E3/E4/E5 | unknown/unproven |

已知开放问题：

- MR capset 8 与 Linux v9 capset 10 不一致；
- `VIRGL_RENDERER_USE_ROCM` 与 `USE_GUEST_VRAM` 都使用 bit 14；
- `vhsakmt_device_get_blob()` 未初始化 `blob->map_info`；
- multi-event wait 返回 `-ENOSYS`；
- 可见 fence 只显式 retire ring 0；
- QEMU RFC v4 不能直接配 Linux v9；
- KFD/virtio 需求正向 amdgpu DRM SVM helper 演进。

固定本地内核中：

- resource blob/context init/blob alignment implemented；
- 本文讨论的 virtio-gpu userptr UAPI 未合入。

文档：

- [社区与机制](virtio-gpu/userptr-rocm-compute.md)
- [ROCm 源码跟踪](virtio-gpu/rocm-source-trace.md)

## 9. Virtio-GPU 迁移状态

固定 QEMU 源码：

- 2D virtio-gpu 有 resource pixel/scanout VMState；
- blob 有 VMState subsection；
- large blob 可注册 migration blocker；
- VirGL 注册 `"virgl is not yet migratable"`；
- Rutabaga 注册 `"rutabaga is not yet migratable"`。

因此：

```text
2D VMState
  != all virtio-gpu variants migratable
  != Host renderer state complete
```

文档：

- [生命周期与迁移](ai-gpu/vm-lifecycle-migration.md)

## 10. KFD/CRIU 证据

固定 Linux UAPI/source：

- `include/uapi/linux/kfd_ioctl.h`
- `drivers/gpu/drm/amd/amdkfd/`

KFD version 快照：

```text
MAJOR 1
MINOR 23
```

可证明：

- process/PASID/device VM；
- compute/AQL/SDMA queue；
- doorbell/event；
- VRAM/GTT/userptr/dma-buf；
- SVM/MMU notifier/migration；
- CRIU `PROCESS_INFO/CHECKPOINT/UNPAUSE/RESTORE/RESUME`；
- MQD checkpoint/restore。

不能直接证明：

- ROCm full-stack virtio ABI 已稳定；
- process CRIU 等于 VM live migration；
- 所有 GPU 都支持 per-queue reset。

文档：

- [KFD/ROCr 源码](ai-gpu/amdkfd-source-trace.md)
- [显存与 SVM](ai-gpu/gpu-memory-semantics.md)
- [Intel Xe SVM 源码](ai-gpu/intel-xe-svm-source-trace.md)
- [生命周期](ai-gpu/vm-lifecycle-migration.md)

## 11. VFIO/SR-IOV/mdev 证据

公开 Linux 文档：

- `Documentation/driver-api/vfio.rst`
- `Documentation/driver-api/vfio-mediated-device.rst`
- `Documentation/PCI/pci-iov-howto.rst`
- `Documentation/PCI/pcieaer-howto.rst`
- `include/uapi/linux/vfio.h`
- `Documentation/userspace-api/iommufd.rst`
- `include/uapi/linux/iommufd.h`
- `Documentation/virt/kvm/x86/{amd-memory-encryption,intel-tdx}.rst`
- `Documentation/arch/x86/tdx.rst`
- `Documentation/driver-api/pci/p2pdma.rst`
- `Documentation/gpu/amdgpu/{xgmi,ualink}.rst`
- `Documentation/gpu/amdgpu/{ras,debugging}.rst`
- `drivers/gpu/drm/amd/amdkfd/kfd_events.c`
- `drivers/gpu/drm/xe/xe_gt_stats_types.h`
- `drivers/gpu/drm/xe/abi/xe_sigid_abi.h`
- `drivers/gpu/drm/xe/`
- `Documentation/ABI/testing/sysfs-driver-intel-xe-sriov`
- `drivers/gpu/drm/amd/amdgpu/amdgpu_virt*.c`
- `drivers/gpu/drm/amd/amdgpu/mxgpu_*.c`

可证明：

- IOMMU group 是最小 ownership unit；
- ACS/bridge/multifunction 可降低隔离粒度；
- VFIO device cdev/container 模型；
- mdev 的 parent/driver/device 生命周期；
- VFIO migration state/dirty tracking UAPI；
- PCIe error/reset 域。

平台依赖：

- 具体 GPU 是否支持 IOMMU isolation；
- FLR/ATS/P2P；
- VF 数量/profile；
- migration 实现；
- vendor firmware。

文档：

- [平台比较](virtio-gpu/platform-comparison.md)
- [安全与故障域](ai-gpu/gpu-virtualization-security.md)
- [Intel GVT-g 源码跟踪](ai-gpu/intel-gvt-g-source-trace.md)
- [VFIO/IOMMUFD/SR-IOV/mdev 源码跟踪](ai-gpu/vfio-iommufd-sriov-source-trace.md)
- [QEMU VFIO 源码跟踪](ai-gpu/qemu-vfio-source-trace.md)
- [NVIDIA Grace GPU VFIO Variant 源码跟踪](ai-gpu/nvidia-grace-vfio-source-trace.md)
- [ATS/PASID/PRI/IOMMU SVA 源码跟踪](ai-gpu/ats-pasid-pri-source-trace.md)

## 12. MIG/vGPU 证据

本地 Linux source 主要提供 PCI/VFIO/IOMMU 基础，不包含 NVIDIA MIG/vGPU
产品内部实现。

因此文档明确依赖：

- NVIDIA MIG User Guide；
- NVIDIA vGPU User Guide；
- 产品/support matrix；
- 固定 driver/firmware/hypervisor 版本。

不能从 Linux PCI 文档推出：

- MIG partition 内部 isolate 细节；
- vGPU scheduler 内部实现；
- 所有产品都支持 SR-IOV 或 live migration；
- MIG instance 自动成为 VM device。

文档：

- [NVIDIA CUDA/MIG/vGPU](ai-gpu/nvidia-cuda-vgpu.md)
- [Windows/Hyper-V GPU 虚拟化](ai-gpu/windows-hyperv-gpu-virtualization.md)
- [云原生 GPU](ai-gpu/cloud-native-gpu-sharing.md)

## 13. NVIDIA CUDA 证据边界

CUDA runtime/driver/vGPU manager 通常不是公开源码。

本库可做：

- 公开 API/ABI 分层；
- UCX/框架观测；
- 产品文档/SMBIOS、`nvidia-smi`、PCI topology 证据；
- 实测行为。

本库不应虚构：

- `libcuda` 内部源码调用；
- vGPU manager 私有状态机；
- MIG firmware 内部数据结构。

文档：

- [NVIDIA CUDA/MIG/vGPU](ai-gpu/nvidia-cuda-vgpu.md)
- [AI Runtime/API 兼容](ai-gpu/ai-runtime-api-compatibility.md)
- [OpenCL/Level Zero/SYCL](ai-gpu/opencl-levelzero-sycl-virtualization.md)

## 14. NCCL/RCCL 证据

固定源码：

- NCCL `3e46c089...`
- RCCL `57e58688...`

可证明：

- communicator/topology discovery；
- P2P/SHM/NET transport；
- ring/tree graph；
- RCCL 与 ROCm-specific topology 差异。

平台依赖：

- 实际 GPU/NIC/fabric；
- IOMMU/ACS/P2P；
- MIG/VF/vGPU topology；
- 多 VM 网络。

文档：

- [NCCL/RCCL 源码](ai-gpu/nccl-rccl-source-trace.md)

## 15. 历史实验证据

以下文档包含此前运行结果，不表示本轮重新验证：

- `PHASE_0_HOST_QEMU_VIRTIO_GPU_PATH.md`
- `PHASE_1_GUEST_DRM_RESOURCES.md`
- `PHASE_2_QEMU_DEVICE_BOUNDARY.md`
- `PHASE_3_VENUS_VULKAN_PATH.md`
- `PHASE_4_GFXSTREAM_RUTABAGA.md`
- `KVM_VENUS_DIAGNOSIS.md`
- `GFXSTREAM_NATIVE_WSI_XID69_DIAGNOSIS.md`
- `PERFORMANCE_AND_HARDWARE_BASELINE.md`

使用历史结果时必须记录：

- 当时 commit/软件版本；
- 硬件与驱动；
- 配置；
- 原始 trace/日志位置；
- 结论适用范围；
- 是否被后续状态覆盖。

## 16. 已知未知项

当前没有足够证据做绝对结论的项：

- 本机是否支持可用 GPU SR-IOV/VF；
- 本机是否支持 GPU MIG；
- 任意 NVIDIA vGPU 产品的内部实现和当前支持矩阵；
- userptr/ROCm 最终会合入的 UAPI/version；
- 所有 native DRM backend 的稳定性和性能；
- 各 GPU 的 live migration 支持；
- 不同平台 userptr 连续 coherence；
- 跨 VM P2P/collective 在目标 fabric 的行为。

这些必须保留为 unknown/platform-dependent。

## 17. 上游变化后的复核规则

### 17.1 Linux virtio-gpu 变化

检查：

- `include/uapi/drm/virtgpu_drm.h`
- `include/uapi/linux/virtio_gpu.h`
- `drivers/gpu/drm/virtio/`
- 是否新增 userptr/readonly/alignment/capset；
- cleanup/error path；
- DMA attribute/coherence。

更新：

- userptr/ROCm 状态；
- blob memory；
- API/ABI 兼容；
- 证据等级。

### 17.2 QEMU 变化

检查：

- capabilities/feature negotiation；
- migration blockers；
- `virtio-gpu-rutabaga.c`；
- `virtio-gpu-virgl.c`；
- `vhost-user-gpu.c`、`contrib/vhost-user-gpu/` 与 display protocol；
- native context enable 条件。

vhost-user 外置后端的固定版本调用链和证据边界见
[Vhost-user-gpu 源码跟踪](virtio-gpu/vhost-user-gpu-source-trace.md)。

### 17.3 virglrenderer 变化

检查：

- capset/flag 编号；
- ROCm MR rebase；
- map_info；
- multi-event；
- ring fence retire；
- DRM backend support。

### 17.4 Mesa 变化

检查：

- Venus ring/protocol；
- GFXStream ring/ResourceTracker；
- vdrm/native shim；
- ROCm frontend；
- extension/capability overclaim。

### 17.5 产品/固件变化

检查：

- GPU support matrix；
- vGPU/MIG profile；
- IOMMU/ACS/P2P；
- reset/migration；
- licence/hypervisor。

## 18. 文档依赖图

```text
四主路径宏观
  -> 四主路径源码
  -> 四路径横向

Native DRM overview
  -> Native DRM source trace
  -> 四路径横向扩展对照

平台比较
  -> 显存/执行/安全/lifecycle
  -> Playbook

AI stack
  -> KFD source
  -> API compatibility
  -> performance
  -> cloud-native
  -> NCCL/RCCL
  -> Playbook
```

## 19. 提交前检查

- [ ] 新结论标注 E0-E5 或明确 unknown；
- [ ] RFC/MR/merged/released 用词准确；
- [ ] 记录 commit/version/date；
- [ ] 指向本地源码或原始 URL；
- [ ] 区分源码、文档、历史和运行证据；
- [ ] 未运行的实验没有冒充已运行；
- [ ] 版本依赖结论没有被外推到所有硬件；
- [ ] 新增专题已进入 README/学习路线；
- [ ] 相对链接和 Markdown fence 已检查。

## 20. 入口

- [GPU 虚拟化总索引](README.md)
- [架构选型 Playbook](ai-gpu/gpu-virtualization-design-playbook.md)
- [AI 学习路线](ai-gpu/learning-roadmap.md)
