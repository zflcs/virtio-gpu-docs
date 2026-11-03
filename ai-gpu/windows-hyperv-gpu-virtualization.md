# Windows/Hyper-V GPU 虚拟化：DDA、GPU-P、WDDM 与 WSL

## 1. 文档定位与证据边界

本文补齐 Windows/Hyper-V 生态中的 GPU 虚拟化方案：

- Discrete Device Assignment（DDA）；
- GPU Partitioning / GPU-PV（通常简称 GPU-P）；
- RemoteFX vGPU 历史方案；
- WDDM 驱动和执行模型；
- WSL2 的 GPU paravirtualization；
- DirectML、CUDA、OpenCL、Vulkan 在 Windows/WSL 中的 API 边界。

对应通用比较：

- [GPU 虚拟化平台比较](../virtio-gpu/platform-comparison.md)
- [AI Runtime、API 与虚拟化兼容性](ai-runtime-api-compatibility.md)
- [OpenCL、Level Zero、SYCL 与 AI compute 虚拟化边界](opencl-levelzero-sycl-virtualization.md)

证据边界：

| 内容 | 证据等级 |
| --- | --- |
| DDA/GPU-P 的产品行为、支持矩阵 | Microsoft 与 GPU 厂商公开文档，E0/E1 |
| WDDM API/驱动模型 | Microsoft WDDM 公开文档 |
| Windows kernel、Hyper-V 和 GPU vendor driver 内部实现 | 通常闭源，未知 |
| 某 GPU/profile/VM 组合可用性 | 固定版本运行证据，E5 |

本文不运行 Hyper-V VM、WSL GPU、DDA、GPU-P 或 AI workload。

不能从产品名推断底层一定使用 mdev、SR-IOV、MIG、VFIO 或 virtio-gpu。

## 2. 先区分四个方案

| 方案 | Guest 设备视图 | 物理 GPU 共享 | 主要边界 |
| --- | --- | --- | --- |
| DDA | 物理 PCI GPU | 通常独占 | Hyper-V + IOMMU + vendor driver |
| GPU-P | Windows 虚拟 GPU/partition | 可多 VM 共享 | WDDM/GPU-PV + vendor driver/manager |
| RemoteFX vGPU | 历史远程虚拟 GPU | Host 图形复制 | 已过时的旧产品栈 |
| WSL2 GPU | Linux 环境中的虚拟 GPU 接口 | Host Windows GPU | Microsoft GPU-PV + vendor user-mode stack |

这些不能混为一谈：

```text
DDA != GPU-P
GPU-P != WSL
WDDM != 一个虚拟化硬件机制
DirectML != CUDA
```

## 3. DDA：物理 GPU 分配给 VM

### 3.1 模型

```text
Physical Windows GPU
  -> Hyper-V dismounts device from normal host use
  -> assign PCI function to VM
  -> Guest loads vendor GPU kernel driver
  -> Guest APIs use physical GPU
```

Guest 看到的是物理 PCI device/BAR/MSI-X，而不是虚拟 GPU protocol。

### 3.2 优点

- Guest 通常可以加载完整 GPU vendor driver；
- 原有 CUDA/ROCm/Vulkan/OpenCL/DirectML 栈最容易保持；
- 不需要 Host 解析 GPU command；
- 单 VM 独占时资源边界相对直观。

### 3.3 限制

- Host 通常失去该 GPU 的普通显示/compute 使用；
- 一卡通常只能给一台 VM；
- 需要 GPU、主板、firmware、Hyper-V 和 IOMMU 满足支持条件；
- 传统 DDA 通常不适合 live migration、checkpoint 或 guardrail 场景；
- GPU reset、display ownership、vendor firmware 和电源管理仍需验证；
- Windows 驱动/Windows 版本可能要求特定虚拟化支持声明。

### 3.4 与 Linux VFIO 的对应

相同点：

- Guest vendor driver；
- 物理 PCI function；
- IOMMU 保护 DMA；
- Host 独占；
- reset/FLR 与 firmware 是故障域。

不同点：

- Hyper-V 的 device assignment 管理和 IOMMU 暴露不由 Linux VFIO 实现；
- Windows Guest 的 WDDM stack 与 Linux vendor KMD/UMD 不同；
- DDA 的迁移/快照策略受 Windows/Hyper-V 产品限制。

## 4. GPU-P/GPU-PV：厂商参与的 GPU 分区

### 4.1 模型

公开材料通常把 GPU-P 描述为：

```text
Physical GPU
  -> Windows GPU partition manager + vendor GPU driver
  -> one or more virtual GPU partitions
  -> Hyper-V VM
  -> Guest WDDM stack
```

GPU-P 不是 PCIe SR-IOV 的同义词。其底层可能由：

- vendor driver 的 software scheduling；
- hardware/firmware partition；
- paravirtualized WDDM interface；
- 专门的管理和资源分配机制；

组合实现。具体 GPU 代际通常决定真实机制，必须以 support matrix 为准。

### 4.2 为什么需要 GPU-P 层

虚拟机不能安全地直接让多个 Windows Guest 任意写同一物理 GPU 的全局寄存器、
调度队列和显存管理状态。GPU-P 层至少需要协调：

- VM/process 与 Host GPU adapter 的身份；
- allocation 和 residency；
- GPU command/DMA buffer；
- fence/sync；
- submit/完成；
- display/copy resource；
- reset 和 partition teardown。

公开文档不会完整展示 Microsoft/vendor 私有命令和 scheduling 数据结构。

### 4.3 资源“partition”表示什么

GPU-P partition 可能表达：

- adapter/device identity；
- VRAM/GTT 或 memory budget；
- engine/scheduling share；
- encode/decode capability；
- display/console path；

不能默认每个 partition 都有独立 HBM 控制器、L2、copy engine 或 reset domain。

### 4.4 支持条件

部署前必须核对：

- 支持 GPU-P 的 Windows 版本和 SKU；
- GPU vendor/model/driver；
- Guest OS 版本；
- VM generation/firmware；
- 是否支持多个 partition；
- VRAM/engine 配额；
- 是否允许同时 Host 使用；
- 是否支持 checkpoint/migration；
- licence 与 support contract。

## 5. WDDM：Windows GPU 栈

### 5.1 通用软件栈

```text
Application
  -> Direct3D/DirectML/OpenCL/Vulkan/CUDA vendor API
  -> vendor user-mode driver (UMD)
  -> D3DKMT / WDDM interfaces
  -> DirectX graphics kernel subsystem
       -> video memory manager
       -> GPU scheduler
       -> display/command submission
  -> vendor kernel-mode driver (KMD/miniport)
  -> physical or virtual GPU
```

在 GPU-P 虚拟化中，UMD/KMD、Host partition manager 和 physical GPU 之间还
存在 paravirtual interface。其精确对象布局由 Microsoft/vendor 定义。

### 5.2 关键对象边界

| 对象 | 身份与资源 |
| --- | --- |
| adapter/device | Guest 可见 GPU |
| context | GPU 工作状态 |
| allocation | video memory 生命周期 |
| residency | 当前在 VRAM/system memory 的位置 |
| command/DMA buffer | Host GPU 可执行内容 |
| sync object/fence | CPU/GPU/engine 依赖 |
| display source/scanout | 桌面和呈现 |

### 5.3 WDDM TDR

Windows Display Driver Model 使用 GPU scheduler、fault detection 和 recovery
机制处理超时/hang。TDR resets 通常影响整个 adapter/device 视图。

多 VM 共享 GPU 时，一个 Guest 触发 reset 会不会影响其他 partition，必须看
GPU-P/vendor 实现，不能只看 Guest 自己的 WDDM 状态。

## 6. GPU-P 控制面与数据面

### 6.1 Control plane

可能需要协调：

- GPU partition 创建/销毁；
- Guest adapter 配置；
- VRAM/engine budget；
- device reset；
- driver/firmware 匹配；
- 物理 GPU 的 Host driver 生命周期；
- VM 生命周期。

某些管理命令是 PowerShell/Hyper-V API，但底层协议未公开。

### 6.2 Data plane

公开可推理的高层路径：

```text
Guest application
  -> Guest UMD
  -> Guest WDDM submission
  -> GPU-PV boundary
  -> Host GPU driver/partition manager
  -> GPU queue
```

不能据此断言每个 API 调用都经过 Windows kernel round-trip；GPU-bound 大
command 可批处理，小 kernel 的 submit/sync 成本更敏感。

### 6.3 Memory plane

需要回答：

- 谁分配 VRAM；
- Guest visible budget 与 physical VRAM 的关系；
- Host system memory/guest RAM 是否可被 GPU DMA；
- residency/eviction 由 Guest 还是 Host 决定；
- shared allocation/copy 是否走 dma-buf 类似机制；
- GPU VA 是 Guest 虚拟、Host 虚拟还是硬件 partition 独立。

公开层通常只能获得对象和性能行为，不能看到全部私有 PTE 格式。

## 7. GPU-P 与 SR-IOV/mdev/vGPU 的关系

### 7.1 不是 SR-IOV

SR-IOV 的必要证据：

- PCI SR-IOV Extended Capability；
- PF/VF；
- VF PCI function；
- PF provisioning；
- Guest 看到 VF。

GPU-P 可以由厂商 driver 实现分区语义，不需要 Guest 看到 VF。

### 7.2 不是 Linux mdev

mdev 是 Linux mediated device framework；Windows GPU-P 使用自己的 Hyper-V/
WDDM/vendor 管理栈。

两者都可能有“一个物理 GPU 的多个虚拟设备”，但 framework、接口和 isolation
实现不同。

### 7.3 不是 NVIDIA vGPU 产品的同义词

NVIDIA vGPU 是特定厂商产品栈，Windows GPU-P 是 Windows 平台机制/产品能力。
某 NVIDIA GPU 的 Windows GPU-P 支持与 Linux vGPU 支持是两个独立矩阵。

### 7.4 是否会组合

产品可能在底层组合 SR-IOV/VF、firmware partition 和 GPU-PV。必须查看具体
release note，不能由接口名称反推。

## 8. RemoteFX vGPU 历史边界

RemoteFX vGPU 是较早的 Windows 远程图形虚拟化方案，目标包括：

- 多 VM 共享 Host GPU；
- Host 编码和远程呈现；
- Guest 图形 API 转发/加速。

现代设计不应默认依赖 RemoteFX：

- 它在 Microsoft 新版本中已被淘汰或移除；
- 具体版本移除点必须查当前文档；
- 旧论文/部署脚本不能作为新产品依据。

其历史价值是展示“图形命令/显示 remoting”与“Guest 物理驱动直通”之间的差异。

## 9. WSL2 GPU

### 9.1 WSL2 是 VM

WSL2 Linux 环境运行在轻量虚拟机中。GPU 加速不是简单把 Windows COM/UMD
直接暴露给 Linux ELF；Windows 提供 GPU paravirtualization 接口。

常见高层路径：

```text
Linux CUDA/DirectML/vendor API
  -> WSL user-mode runtime/driver
  -> /dev/dxg or Microsoft GPU-PV interface
  -> Windows GPU-PV
  -> Host Windows WDDM driver
  -> physical GPU
```

`/dev/dxg` 和 DirectX kernel/user components 属于公开可观察接口边界。
内部 Host command、队列和 scheduling 格式不是本文可证明的源码事实。

### 9.2 CUDA

NVIDIA 支持在 WSL 中运行 CUDA workload，但通常依赖专用 WSL driver library
和 Windows Host driver，而不是在 WSL 内加载完整 Windows GPU KMD。

必须核对：

- Windows NVIDIA driver；
- WSL kernel；
- WSL CUDA driver package；
- CUDA toolkit；
- framework 版本；
- GPU architecture；
- WSL GPU-P 支持矩阵。

### 9.3 Vulkan/OpenCL/DirectML

不同 API 在 WSL 中成熟度不同：

- DirectML 是 Windows API 生态；
- Vulkan 可通过 WSLg 或 GPU-PV 支持，具体版本/extension 依实现；
- OpenCL 依赖 vendor WSL runtime；
- CUDA 不是 Vulkan/OpenCL 的替代层；
- 不能因为 CUDA 可用就断言 OpenCL/Level Zero 可用。

### 9.4 与 virtio-gpu 的区别

WSL GPU-PV 与 Venus/GFXStream 都能让 Linux Guest 访问 Host GPU，但：

- Venus/GFXStream 有开放的 virtio-gpu capset、ring 和 renderer 源码；
- WSL GPU-PV 主要通过 Windows 闭源 GPU-PV/DirectX 栈；
- API、device node、memory 和 sync ABI 不同；
- 两者不能交换 Guest driver/binary。

## 10. AI API 映射

| API/栈 | Windows | WSL2 | 关键兼容条件 |
| --- | --- | --- | --- |
| CUDA | vendor driver | NVIDIA WSL driver stack | GPU/driver/toolkit 矩阵 |
| DirectML | Windows API | 由框架/运行时支持情况决定 | WDDM/GPU-PV、DML runtime |
| Vulkan compute | vendor Vulkan ICD | WSLg/GPU-PV Vulkan | ICD/extension |
| OpenCL | vendor Windows ICD | WSL vendor runtime | ICD/backend |
| HIP/ROCm | 依 AMD 产品支持 | 依 ROCm/Windows/WSL 支持 | 通常不能假设与 Linux 等价 |
| oneAPI/Level Zero/SYCL | 依 Intel 支持 | 依 WSL backend | 无通用自动兼容 |

### 10.1 PyTorch

Windows/WSL 中 PyTorch backend 可能依赖：

- CUDA；
- ROCm where supported；
- DirectML extension；
- Intel XPU/oneAPI；
- CPU。

选择 backend 后，caching allocator、custom extension、distributed backend 和
TorchScript/compile 目标都会重新形成兼容矩阵。

### 10.2 推理服务

Windows 原生 AI serving 与 WSL Linux serving 的差异不只在 GPU：

- IPC/network；
- signal/process model；
- filesystem；
- NUMA；
- container/Kubernetes；
- GPU telemetry。

DDA/GPU-P 只是设备执行层。

## 11. 隔离与安全

### 11.1 DDA

- 主要依赖物理 GPU 独占、Hyper-V device assignment 和 IOMMU；
- Guest vendor driver/firmware 暴露面更大；
- reset/firmware 故障可能影响整 GPU；
- Host 不再使用 GPU 降低共享攻击面，但增加可用性风险。

### 11.2 GPU-P

- Host GPU driver/partition manager 是共享信任边界；
- Guest WDDM/UMD command、allocation 和 sync 进入 Host；
- VRAM/cache/engine 可能共享；
- side channel、fault containment、reset scope 由实现决定；
- privileged Hyper-V 管理角色仍是高价值目标。

### 11.3 WSL

- WSL 是 Host 管理下的轻量 VM；
- Linux workload 与 Windows Host 共享 GPU；
- `/dev/dxg` 接口和 Microsoft/vendor user-mode stack 是攻击面；
- WSL 方便不等于 VM-level isolation 或 confidential GPU。

## 12. 生命周期与迁移

| 操作 | DDA | GPU-P | WSL |
| --- | --- | --- | --- |
| VM snapshot | 通常不支持/受产品限制 | 依 Windows/vendor | 不是传统 VM snapshot |
| live migration | 通常不支持 | 依产品 support matrix | 不适用 |
| save/restore | 物理设备状态复杂 | partition/Host state 复杂 | 依进程重启 |
| GPU reset | Guest/物理 device reset | partition 与 Host manager 协调 | Windows Host reset |
| driver update | Host 解绑 + Guest driver | Host PF/manager 与 Guest 同时升级 | Host/WSL driver 同步 |

任何“支持迁移”必须核对：

- Windows/Hyper-V 版本；
- GPU/vGPU/partition profile；
- Host/Guest driver；
- firmware；
- 网络/storage；
- source/destination 兼容性；
- workload checkpoint。

## 13. 与 virtio-gpu、VFIO、Xe SR-IOV 对照

| 维度 | Windows DDA | Windows GPU-P | Windows WSL GPU | virtio-gpu | Linux VFIO |
| --- | --- | --- | --- | --- | --- |
| Guest device | physical PCI GPU | virtual GPU/partition | WSL GPU-PV | virtio-gpu | physical PCI/VF |
| Guest driver | vendor KMD/UMD | WDDM GPU-P stack | WSL user stack | virtio KMD + UMD | vendor KMD |
| Host command ownership | Guest/physical GPU | Host GPU-P/vendor manager | Windows Host GPU-PV | renderer/vendor driver | Guest/physical GPU |
| Multi-VM | 低 | 可 | WSL 与 Host 共享 | 可 | 依 physical/VF |
| Main boundary | Hyper-V + IOMMU | GPU-PV/WDDM/vendor | GPU-PV | virtqueue + parser | IOMMU + PCI |
| Source visibility | 部分 OS API | 大量闭源 | 大量闭源 | 多数开源路径 | 多数 Linux 开源 |

## 14. AI workload 选型

### 14.1 单租训练

DDA 或受支持的 GPU-P profile 更适合保留完整 CUDA/HIP 栈。DDA 的可用性和
迁移限制需要在采购/部署前确认。

### 14.2 多 VM 推理

GPU-P 可提高密度，但必须测量：

- VRAM quota；
- engine fairness；
- small-kernel submit latency；
- P99 干扰；
- partition reset 影响；
- Host 与 Guest telemetry。

### 14.3 WSL 开发

WSL 适合开发/验证在 Linux 工具链中的 CUDA/PyTorch workload，但不能把开发
便利当作生产 GPU 隔离、migration 或多租安全。

### 14.4 Windows 原生 AI

DirectML/CUDA/OpenCL 等 backend 选择需要和目标框架支持矩阵匹配。Windows
服务、driver service 和 session 0 行为也会影响挂起/恢复。

## 15. 失败矩阵

| 症状 | 优先检查 |
| --- | --- |
| DDA 设备无法 dismount | Host driver/device、Hyper-V 支持矩阵 |
| Guest 看不到 DDA GPU | PCI assignment、MMIO、IOMMU、Guest driver |
| DDA GPU 能枚举但 compute 失败 | firmware/UEFI、driver、display/reset |
| GPU-P partition 创建失败 | Windows SKU、GPU/driver、partition budget |
| Guest GPU 可枚举但 VRAM 小 | partition profile/quota |
| 多 VM 性能互相干扰 | engine/HBM/cache 共享，不能只看 VRAM |
| GPU-P reset 影响他 VM | Host partition manager/firmware |
| WSL `/dev/dxg` 不存在 | Windows Host GPU driver、WSL kernel/driver |
| WSL CUDA 不可用 | NVIDIA WSL driver/toolkit 版本矩阵 |
| WSL 能 CUDA 但不能 OpenCL | vendor/API support matrix |
| Windows 暂停/恢复后 GPU 失败 | WDDM/driver/partition lifecycle |

## 16. 专家审查清单

### 16.1 设备

- Guest 看到 physical PCI、virtual GPU 还是 WSL dxg？
- 是独占 DDA 还是共享 GPU-P?
- GPU model、SKU、driver、firmware、Windows/Hyper-V 版本是什么？

### 16.2 执行

- Guest UMD/KMD 与 Host GPU driver 谁是 command owner？
- queue/submit 在哪里跨 VM 边界？
- 小 kernel、graph replay、copy 和 collective 的路径分别是什么？

### 16.3 内存

- VRAM budget、residency、paging 和 eviction 由谁决定？
- Guest RAM 如何 DMA？
- shared allocation 是否跨 VM 安全？

### 16.4 生命周期

- partition/DDA device 如何创建/销毁？
- GPU reset/TDR/FLR 影响哪个 scope？
- migration/snapshot 是否真正支持？

### 16.5 AI

- CUDA/DirectML/OpenCL/Vulkan 哪个是实际 backend？
- PyTorch/custom extension/distributed backend 是否匹配？
- 是否仍运行在 target Windows/WSL driver/firmware 上？

## 17. 检查题

1. DDA 与 Linux VFIO passthrough 的共同点和平台差异是什么？
2. GPU-P 为什么不等于 PCIe SR-IOV？
3. WDDM UMD、KMD、video memory manager 和 scheduler 各是什么层？
4. GPU-P 的 partition 为什么不保证独立 HBM/L2/reset domain？
5. 多 VM 共享 GPU 时 TDR/reset scope 为什么必须验证？
6. WSL2 为什么不能被描述为“直接把 Windows GPU 给 Linux”？
7. WSL CUDA 为什么不能自动推出 OpenCL/Level Zero 兼容？
8. DDA 为什么通常不利于 live migration？
9. GPU-P 的 support matrix 需要记录哪些版本？
10. GPU-P 与 NVIDIA vGPU 产品为什么不能混用支持结论？
11. DirectML 与 CUDA 在兼容性上差在哪？
12. 如何验证多租推理的 P99 和 VRAM 隔离？
13. 为什么 Windows Host 可用 GPU workload 不代表 WSL 可用？
14. GPU-PV 与 virtio-gpu/Venus 的 Guest ABI 为什么不同？
15. 需要哪些固定版本运行证据才能判定生产可用？

## 18. 与其他专题

- [GPU 虚拟化平台比较](../virtio-gpu/platform-comparison.md)
- [AI Runtime、API 与虚拟化兼容性](ai-runtime-api-compatibility.md)
- [OpenCL、Level Zero、SYCL 与 AI compute 虚拟化边界](opencl-levelzero-sycl-virtualization.md)
- [NVIDIA CUDA、MIG 与 vGPU 架构边界](nvidia-cuda-vgpu.md)
- [GPU 虚拟化的隔离、安全与故障域](gpu-virtualization-security.md)
- [GPU VM 生命周期、热插拔、Checkpoint 与迁移](vm-lifecycle-migration.md)
- [云原生 AI 中的 GPU 共享与虚拟化](cloud-native-gpu-sharing.md)
