# AI GPU 互连与虚拟化 Fabric

## 1. 文档定位

多 GPU AI 性能不只由单卡算力决定。模型并行、数据并行、tensor parallel、
expert parallel、推理 prefill/decode disaggregation 和 checkpoint 都依赖
GPU/NIC/CPU 之间的数据路径。

本文连接：

- PCIe P2P；
- NVLink/NVSwitch；
- AMD XGMI hive；
- AMD UALink/NPA；
- RDMA/RoCE/InfiniBand；
- CXL/coherent fabric；
- IOMMU/ACS；
- VM/VF/MIG/vGPU topology；
- NCCL/RCCL transport。

本文不运行互联或 collective 实验。

## 2. 互连层级

### 2.1 Intra-GPU

- SM/CU -> L2 -> HBM；
- 多 XCD/partition；
- GPU internal fabric；
- copy engine；
- display/media engine。

不属于本文章重点，但会影响 HBM/occupancy。

### 2.2 Intra-node GPU scale-up

- PCIe P2P；
- NVLink/NVSwitch；
- XGMI/UALink；
- CXL fabric。

目标是 GPU 直接 load/store 或 DMA peer memory。

### 2.3 Intra-node GPU scale-out

- GPU -> NIC；
- GPU Direct RDMA；
- PCIe P2P 到 NIC；
- CPU bounce memory；
- peer-to-peer memory windows。

### 2.4 Inter-node

- InfiniBand/RoCE；
- TCP/QUIC；
- NVLink network/其他 scale-out；
- network switching 和 congestion control。

### 2.5 Collective software

```text
application
  -> PyTorch/JAX process group
  -> NCCL/RCCL communicator
  -> topology graph
  -> P2P / SHM / NET / NVLS channels
  -> GPU/NIC fabric
```

## 3. PCIe P2P

### 3.1 原理

PCIe TLP 在 switch 内可按 ACS 配置直接路由 peer device，而不经过 Root Complex
或 Host memory。

Linux
[`p2pdma.rst`](../../linux/Documentation/driver-api/pci/p2pdma.rst:1)
指出：

- switch 内 P2P 与 ACS 配置相关；
- Host bridge/root port 间转发不一定由 PCIe 定义；
- 内核默认阻止跨 hierarchy domain routing；
- 允许列表可用于已知安全硬件；
- provider/client/orchestrator 必须协作管理生命周期；
- P2P memory 可能是 MMIO，没有普通 `struct page`；
- remove 前必须撤销所有 DMA/CPU/PTE mappings；
- dma-buf revocation 是另一种安全生命周期机制。

### 3.2 AI 含义

P2P 可以提供：

- 低延迟 GPU-GPU copy；
- GPU-NIC direct DMA；
- 减少 Host bounce/copy。

但：

- ACS/IOMMU 可能禁用；
- VM 内 Guest bus ID 不等于 Host物理路径；
- VF/MIG 组合可能不允许 peer；
- P2P provider 生命周期复杂；
- P2P 不等于 coherent，同步仍由框架负责。

## 4. NVIDIA NVLink/NVSwitch

NVIDIA 的 NVLink/NVSwitch 内部协议和 topology discovery 不是完整开源 Linux
UAPI。公开证据主要来自：

- CUDA/NVML peer API；
- NCCL topology graph；
- `nvidia-smi topo -m` 输出；
- support docs；
- PCI/NVSwitch topology files；
- 实测 bandwidth。

### 4.1 Virtualization effects

- Guest 可能只看见 GPU，不看见完整 NVSwitch topology；
- vGPU/MIG profile 可能限制或隐藏 P2P；
- NCCL topology XML 可能需 Host 注入；
- VFIO whole GPU 更接近真实 topology；
- 跨 VM NVLink 共享不是默认能力。

### 4.2 NCCL transport

NCCL 会检查 NVML/CUDA P2P、PCI/NVLink 距离和 network GDR capability，再
选择 P2P/SHM/NET/NVLS 等。

代码路径见
[NCCL/RCCL 源码跟踪](nccl-rccl-source-trace.md)。

## 5. AMD XGMI

### 5.1 Hive

[`amdgpu_xgmi.c`](../../linux/drivers/gpu/drm/amd/amdgpu/amdgpu_xgmi.c:328)
说明 XGMI 把多 GPU 组成 homogeneous memory space，由：

- 64-bit hive ID；
- 64-bit node ID；
- per-node VRAM segment。

组织。

### 5.2 Topology evidence

sysfs `xgmi_hive_info` 提供：

- hive ID；
- node device ID；
- physical ID；
- hop count；
- link count；
- port number；
- error。

### 5.3 Reset domain

源码说明：

- bare metal/passthrough 可创建 XGMI_HIVE reset domain；
- SRIOV VF 由 Host 决定 FLR/chained reset，Guest收到 host notification；
- hive 中设备可能需要在 topology update 后重新同步 reset。

这证明 GPU P2P topology 与 reset 域可能关联。

### 5.4 RCCL

RCCL 扩展 NCCL topology：

- XGMI edge；
- `ncclTopoAddXGMI()`；
- Rome/GIO parser；
- HIP/NVML-equivalent device query。

Guest VF/MIG/VIOMMU 不会自动暴露真实 XGMI hive。

## 6. AMD UALink/NPA

[UALink 文档](../../linux/Documentation/gpu/amdgpu/ualink.rst:1) 给出一个
scale-up fabric 的源码级设计：

### 6.1 NPA

- NPA = Network Physical Address space；
- contiguous physical address space partitioned per GPU；
- each GPU has page tables for its NPA space；
- remote GPU direct load/store；
- CPU cannot access remote memory through this path。

### 6.2 Export/import

Exporter:

- creates opaque UALink handle for GEM；
- exported memory pinned；
- remote exporter TLB shootdown on free。

Importer:

- imports opaque handle；
- queries NPA address；
- creates GEM/dma-buf；
- maps NPA into local per-process GPUVM。

### 6.3 Lifecycle

- NPA addresses exchanged between drivers, not userspace；
- remote peer reset/offline cleans lost references；
- dma-buf revocation can invalidate importer.

### 6.4 AI implications

This is a hardware coherent/scale-up-like path, not network transport. But:

- isolation depends on authorization;
- remote TLB shootdown/lifetime;
- peer fault/reset;
- GPUVM integration;
- VM exposure may hide/partition NPA;
- multi-tenant sharing needs additional policy.

## 7. RDMA / GPU Direct

### 7.1 Two domains

```text
GPU scale-up fabric (NVLink/XGMI/UALink)
  vs
GPU-to-NIC scale-out fabric (PCIe P2P -> RDMA)
```

RDMA GDR often requires:

- NIC support；
- GPU memory registration；
- peer/PCI topology；
- IOMMU/P2P permission；
- MR/keys/QP lifecycle；
- network rail selection。

### 7.2 Virtualization

VM:

- Guest sees virtual NIC；
- GDR may be unsupported;
- Host network provider may proxy;
- physical GPU/NIC topology hidden;
- P2P/IOMMU domain crossing.

Fallback:

```text
GPU -> Host/guest memory -> NIC DMA
```

功能正确但增加 copy/latency。

## 8. CXL and coherent fabrics

CXL 可提供：

- memory expansion；
- cache coherent host-device path；
- pooling/partitioning。

对 GPU/AI：

- potential accelerator memory pool；
- CPU/accelerator coherent view；
- different latency/bandwidth from HBM/NVLink；
- virtualization/multitenancy roadmap varies。

不能因为设备“支持 CXL”就认为：

- GPU HBM 可直接 coherent；
- P2P 自动可用；
- VM assignment/migration 自动支持。

## 9. Fabric topology graph

### 9.1 Nodes

- GPU；
- GPU partition/MIG/VF；
- PCIe switch/port；
- CPU socket/NUMA；
- NVSwitch；
- XGMI node/hive；
- UALink node；
- NIC；
- network switch；
- remote host。

### 9.2 Edges

- NVLink；
- XGMI；
- UALink；
- PCI；
- C2C；
- SYS/memory；
- NET；
- peer memory authorization。

### 9.3 Attributes

- bandwidth；
- latency；
- hop count；
- coherence；
- DMA/P2P capability；
- read/write permission；
- reset domain；
- trust/security domain。

## 10. Virtualization impact

### 10.1 VFIO

Guest 通常看见完整 GPU PCI topology，但：

- Guest bus ID 是虚拟/remapped；
- Host IOMMU/ACS 仍决定可达；
- Guest 与 Host topology XML 可能不同；
- 多 GPU assignment 的 SLIT/ACPI 需正确。

### 10.2 SR-IOV

- Guest sees VFs；
- peer/NVLink/XGMI capability 依 PF/profile；
- P2P domain likely VF-specific；
- collective may fallback to SHM/NET。

### 10.3 MIG

- single physical GPU partition；
- P2P between instances profile-dependent；
- MIG device UUID/UUID topology；
- not NVSwitch/compute fabric partition automatically。

### 10.4 vGPU

- Guest vendor device；
- NVLink/topology may be hidden;
- remote display/network channels;
- product-specific P2P/GDR matrix。

### 10.5 Virtio/API remoting

- Guest only sees virtio/resource IDs；
- physical GPU/NIC P2P inaccessible;
- Host renderer may run multiple local GPU processes;
- collective across VMs normally network-only;
- ROCm proxy needs explicit peer/topology protocol.

## 11. P2P lifecycle

For direct remote memory:

```text
allocate
  -> export/authorize
  -> pin
  -> import
  -> map peer PTE
  -> use with fence
  -> unmap/TLB shootdown
  -> revoke
  -> free
```

Missing TLB shootdown can leave stale remote PTE writing freed memory.

Missing revocation can let a removed peer continue DMA.

## 12. Security

- P2P can bypass Root Complex/IOMMU depending topology/ACS;
- exported memory must be authorized;
- opaque handles must not be guessable enough to replace ownership checks;
- remote reset must revoke access;
- VM/VF partition must not expose sibling physical addresses;
- confidential fabric needs per-link encryption/trust;
- topology discovery can leak tenant placement;
- GDR MR keys must be lifecycle safe.

See [PCIe/IOMMU/ATS/P2P](pcie-iommu-ats-p2p.md) and
[Confidential GPU](confidential-gpu-virtualization.md).

## 13. Performance

### 13.1 Latency factors

- hop count；
- switch congestion；
- P2P vs host bridge；
- copy engine vs direct load/store；
- message size；
- collective algorithm；
- rank skew；
- DRAM/HBM bandwidth。

### 13.2 Small messages

For decode/all-reduce small tensor:

- launch/setup latency；
- transport handshake；
- fence；
- network RTT；
- congestion。

Large message bandwidth not representative.

### 13.3 Large messages

- link bandwidth；
- switch bisection；
- error correction；
- memory registration；
- GPU memory copy engines。

## 14. Failure matrix

| Symptom | Check |
| --- | --- |
| P2P enable fails | ACS/IOMMU/profile/P2P query |
| guest sees fewer GPUs | PCI/topology/vGPU/MIG mapping |
| all-reduce hangs | rank topology, peer authorization, network |
| fallback slow | transport selected, GDR/SHM/NET |
| stale remote write | TLB shootdown/revocation |
| one peer reset kills hive | reset domain/firmware |
| CXL memory slower than expected | latency/coherence/topology |
| migration topology mismatch | destination PCI/NVLink/XGMI |

## 15. Evidence checklist

1. Guest-visible GPU/PF/VF/MIG attributes.
2. Host physical GPU (BDF, serial, node).
3. PCIe switches/root ports/ACS.
4. NVLink/XGMI/UALink topology.
5. NIC and rail.
6. IOMMU/ATS/PASID.
7. P2P enable/authorization.
8. NCCL/RCCL topology XML.
9. transport/channel selection.
10. bandwidth/latency/P99.
11. reset domain.
12. security/confidential trust.

## 16. Common myths

### “Same Host means P2P”

No. Requires topology, ACS/IOMMU, driver, profile.

### “VF/MIG always exposes full peer fabric”

No.

### “Remote memory load/store means zero-copy collective”

It is direct memory access, but still bandwidth/PTE/TLB/fence limited.

### “RDMA and scale-up fabric are one layer”

No. RDMA involves NIC/QP/MR; NVLink/XGMI/UALink are GPU scale-up paths.

### “Guest bus ID identifies Host physical slot”

No. Virtual/remapped topology.

## 17. Questions

1. PCIe P2P 与 NVLink/XGMI/UALink 的地址/隔离差异是什么？
2. ACS 如何改变 P2P routing？
3. XGMI hive 的 hive ID/node ID 有什么作用？
4. UALink NPA 为什么不对 CPU开放？
5. Export/import memory 为什么必须 pin 和 TLB shootdown？
6. GDR 与 scale-up P2P 在协议上差在哪？
7. VF/MIG/vGPU 为什么可能隐藏物理 topology？
8. 为什么 same-host collective 可能 fallback 到 SHM/NET？
9. P2P 成功后为什么仍要 fence/barrier？
10. Reset domain 如何影响 hive collective？
11. 单卡大 GEMM 为什么不能证明多卡扩展效率？
12. CXL coherent fabric 为什么不自动等于 GPU HBM P2P？

## 18. 与其他专题

- 源码 collective：  
  [NCCL/RCCL](nccl-rccl-source-trace.md)
- PCIe/IOMMU：  
  [PCIe、IOMMU、ATS/PASID 与 GPU P2P](pcie-iommu-ats-p2p.md)
- AMD SR-IOV：  
  [AMD GPU SR-IOV 与 MxGPU](amd-gpu-sriov-mxgpu.md)
- Intel Xe：  
  [Intel Xe GPU 虚拟化与 SR-IOV](intel-xe-sriov.md)
- NVIDIA：  
  [NVIDIA CUDA、MIG 与 vGPU](nvidia-cuda-vgpu.md)
- 安全：  
  [GPU 虚拟化安全](gpu-virtualization-security.md)
- 性能：  
  [GPU 性能与可观测性](gpu-performance-observability.md)
