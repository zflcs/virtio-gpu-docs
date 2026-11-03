# GPU 虚拟化：硬件、软件与 API 分层

本文解释 VFIO、SR-IOV、mdev、MPT、MIG、NVIDIA vGPU 与 virtio-gpu 的关系。
目的不是给出一个“性能排行”，而是建立能评审实际产品的架构模型。

## 1. 六个问题

判断任何 GPU 虚拟化方案，依次问：

1. **Guest ABI**：Guest 看到 virtio-gpu、虚拟厂商设备、SR-IOV VF，还是
   MIG partition？
2. **拦截边界**：Hypervisor 转发 API、驱动 UAPI、MMIO，还是几乎不拦截？
3. **执行所有者**：Guest CPU、Host renderer、Guest vendor driver 还是
   物理 GPU 调度器提交 work？
4. **内存边界**：Guest VA/GPA、Host VA、IOMMU DMA、GPU VA、VRAM 如何对应？
5. **多租户机制**：时间片、软件 context、PCIe function、硬件 partition
   还是整设备独占？
6. **生命周期**：热插拔、FLR/reset、挂起、snapshot、live migration 和故障
   恢复各自支持到哪里？

这六个维度相互独立。同一产品名称可能同时使用多个机制；同一底层机制也可
被不同产品包装。

## 2. 先区分五个抽象层

PCIe Requester ID、IOMMU stage-1/2、IOMMUFD、PASID/PRI、ATS 和 P2P 的详细
源码边界见
[PCIe、IOMMU、ATS/PASID 与 GPU P2P](../ai-gpu/pcie-iommu-ats-p2p.md)。

### 2.1 设备模型层

Guest 看到什么虚拟设备：

- virtio-gpu 虚拟 PCI 设备；
- VFIO 暴露的物理 PCI function；
- mdev 生成的 vendor-defined mediated device；
- SR-IOV VF；
- MIG 实例通过某种 VFIO/vendor 通道暴露。

设备模型决定 Guest 驱动、配置空间、BAR、中断和 hotplug 语义。

### 2.2 命令/API 层

Host 解释什么：

- **API remoting**：OpenGL、Vulkan、GLES 等高层 API；
- **driver/UAPI remoting**：例如 virtio-gpu DRM native context；
- **MMIO/PCI passthrough**：Host 不解释 GPU 命令，Guest 驱动直接写设备。

低层不等于更安全。API remoting 要解析复杂 Guest 数据结构；VFIO 要让 Guest
厂商驱动和硬件/firmware 直接接触；两类方案有不同攻击面。

### 2.3 PCIe 功能层

- PF：Physical Function，管理物理设备和 VF。
- VF：Virtual Function，由 SR-IOV capability 暴露的轻量 PCI function。
- mdev：不是 PCIe SR-IOV 标准 function，而是软件生成并挂到 mdev bus 的
  mediated device。

### 2.4 GPU 内部资源层

- context/time slice：多个软件 context 共享同一 GPU engine。
- SR-IOV VF：硬件/固件为 function 提供配置空间、队列和调度配置。
- MIG：硬件把计算和显存空间切成多个 instance。
- vendor vGPU：产品层组合上述机制并加入 guest driver、manager、
  scheduling、migration 和 licence。

### 2.5 VMM/I/O 基础设施层

- VFIO：Linux 框架，向 VMM 暴露受 IOMMU 保护的设备，并管理 groups/
  regions/interrupts/migration UAPI。
- IOMMU：限制设备 DMA 到被授权 GPA/HPA 范围。
- virtqueue：virtio-gpu 的 control/cursor queue 传输。
- Rutabaga/virglrenderer：QEMU 与 Host graphics backend 的适配层。

VFIO 和 mdev 是机制，不等于某种固定性能或隔离等级。

## 3. Virtio-GPU 属于哪一层

virtio-gpu 是**虚拟设备 + 可插拔 renderer protocol**：

```text
Guest API/runtime
  -> capset-specific protocol
  -> virtio-gpu PCI device
  -> QEMU
  -> renderer
  -> Host GPU
```

| virtio-gpu 路径 | 本质 |
| --- | --- |
| 2D | 设备模拟 + Guest resource/display |
| VirGL | OpenGL API remoting |
| Venus | Vulkan API remoting |
| GFXStream | 多 API streaming，常用于 Android |
| native DRM | 厂商 DRM/命令 ABI remoting |
| ROCm proposal | libhsakmt/HSA-oriented API remoting |

与 passthrough 的根本区别：

- Guest PCI ID 通常是 `1af4:1050`；
- Guest 不拥有物理 GPU BAR 和 MSI-X；
- Host renderer/driver 解释或代理 Guest 请求；
- 多个 Guest 可共享 Host GPU context/renderer；
- Guest 看到的厂商名可以是 Host 能力转发，不代表直通。

virtio-gpu 可以描述为“软件设备边界 + Host 执行后端”，但它不是一个固定
隔离等级。vrend in-process、Venus render server、GFXStream backend 和 Host
libhsakmt 的信任边界都不同。

## 4. VFIO 整设备直通

### 4.1 机制

```text
Guest vendor driver
  -> Guest GPU MMIO / RPC / MSI-X
  -> QEMU VFIO
  -> Host IOMMU
  -> entire physical PCI GPU
```

Linux [VFIO 文档](../../linux/Documentation/driver-api/vfio.rst) 把 VFIO 描述为
device/IOMMU-agnostic framework：向 userspace/VMM 暴露 direct device access，
并用 IOMMU 限制设备 DMA。

部署时通常：

1. 确认 GPU 位于可安全隔离的 IOMMU group；
2. 从 Host 正常驱动解绑；
3. 绑定 `vfio-pci` 或厂商 VFIO driver；
4. QEMU 暴露 config、BAR、ROM、interrupt 和 DMA mapping；
5. Guest 加载真实厂商 KMD/UMD。

### 4.2 优点

- Guest vendor driver 直接编程物理 function；
- 私有 API、CUDA/ROCm、视频、显示等通常最容易保持原生行为；
- 单租户下没有高层 API decode 往返；
- Host 不需要理解每个 GPU command。

### 4.3 风险与限制

- 一个物理 GPU 通常被一台 VM 独占；
- Host 失去该 GPU；
- 必须处理 IOMMU group、PCIe bridge、ACS 和 multifunction backdoor；
- GPU FLR/reset、AER、电源管理和 firmware 故障恢复复杂；
- 某设备可由 IOMMU 保护 DMA，不等于其 firmware、VRAM 隔离或 reset 已被
  证安全；
- live migration 需要设备/驱动实现 VFIO migration UAPI，很多 GPU 不支持或
  只支持厂商特定状态；
- 唯一宿主显示 GPU 的直通实验可能使 Host 失去交互界面。

VFIO 是暴露和隔离机制，不是“设备天然安全”的证明。

## 5. SR-IOV

### 5.1 PCIe SR-IOV

[Linux PCI SR-IOV 文档](../../linux/Documentation/PCI/pci-iov-howto.rst) 的定义：

- PF 是物理 function；
- VF 是由 PCIe SR-IOV Extended Capability 暴露的 virtual function；
- PF 控制 VF 数量；
- 每个 VF 有独立 Routing ID、PCI config space 和 MMIO；
- VF 可作为 PCI 设备热插拔并由正常驱动绑定。

```text
Physical GPU PF
  |-- VF 1 -> VFIO/vendor assignment -> VM A
  |-- VF 2 -> VFIO/vendor assignment -> VM B
  `-- VF N -> VFIO/vendor assignment -> VM N
```

### 5.2 GPU SR-IOV 不只是 PCIe 配额

Intel Xe PF/VF、GuC provisioning、VRAM quota、VF FLR 和 VFIO migration 的
源码案例见
[Intel Xe GPU 虚拟化与 SR-IOV](../ai-gpu/intel-xe-sriov.md)。
AMD VF mailbox、full-access/runtime、RLCG 间接寄存器和 FLR 源码案例见
[AMD GPU SR-IOV 与 MxGPU](../ai-gpu/amd-gpu-sriov-mxgpu.md)。

现代 GPU PF 驱动还管理：

- VF 显存/GTT/GGTT 配额；
- execution quantum；
- preemption timeout；
- scheduling priority；
- VF pause/stop/reset；
- VF state migration。

Linux Intel Xe 的
[`sysfs-driver-intel-xe-sriov`](../../linux/Documentation/ABI/testing/sysfs-driver-intel-xe-sriov)
公开了 `exec_quantum_ms`、`preempt_timeout_us`、`sched_priority`、
`vram_quota` 和 `stop` 等控制。这是重要证据：GPU VF 可能仍由 PF 做时间片
调度，而不是每个 VF 永远拥有完全独立的硬件 engine。

### 5.3 VFIO 与 SR-IOV 的关系

它们经常组合，但不是同义词：

```text
SR-IOV 创建 VF
  -> VFIO 把 VF 安全暴露给 VM
  -> Guest driver 使用 VF
```

也可以用厂商 hypervisor 通道分配 VF，而不经过通用 `vfio-pci`。具体产品要
看实现。

### 5.4 优点与边界

优点：

- 一张物理 GPU 可服务多台 VM；
- Guest 使用真实厂商驱动；
- VF 有硬件/固件参与隔离，通常比共享 Host API context 有更强 boundary；
- 数据路径通常比 API remoting 少高层转换。

边界：

- 只有支持 SR-IOV 的 GPU/固件/PF driver 组合可用；
- VF 数量、VRAM quota、engine、视频/显示能力依 profile；
- PF 必须由可信 Host 管理；
- FLR、错误隔离、ATS/PRI、ACS 和 firmware 行为必须验证；
- Guest/Host VF driver 与 PF manager 版本强耦合；
- VF 迁移能力依硬件、PF migration driver 和 VMM。

“每 VM 一个 VF”不等于“每 VM 一块完全独立物理 GPU”。

## 6. Mediated Device（mdev）

### 6.1 定义

mdev 是 Linux 的 mediated device framework。父设备驱动注册 device type，
管理创建/销毁，并把 vendor-specific device 通过 mdev bus 暴露给
`vfio_device`。

```text
Parent GPU driver
  -> register mdev types/capabilities
  -> user writes a UUID to create
  -> mdev_device appears
  -> VFIO attaches mdev to VM
```

Linux
[`vfio-mediated-device.rst`](../../linux/Documentation/driver-api/vfio-mediated-device.rst)
说明其管理职责包括：

- 创建/销毁 mediated device；
- 加入/移出 mdev bus；
- 加入/移出 IOMMU group；
- 通过 sysfs 提供 `mdev_supported_types`、`create`、`remove` 等接口。

### 6.2 它解决的是“没有现成 SR-IOV 怎么多租”

mdev 的动机是让没有标准 SR-IOV capability 的设备也能使用统一设备管理和
VFIO 接口。厂商驱动可在软件中创建多个虚拟 function 视图。

### 6.3 mdev 不是隔离保证

mdev 是管理/暴露框架，真正的数据路径由 vendor driver 决定：

- 可能是硬件 context/workqueue；
- 可能是 trap-and-emulate；
- 可能是通过 Host renderer 共享 GPU；
- 可能有独立 fault domain，也可能只靠软件 domain。

因此：

```text
mdev ≠ SR-IOV
mdev ≠ MPT
mdev ≠ 一定低性能
mdev ≠ 一定硬件隔离
```

mdev 可以承载 MPT 式设计，也可以承载别的实现。

## 7. Mediated Pass-Through（MPT）

### 7.1 定义

GPU 文献中的 MPT 通常指 **Mediated Pass-Through**：Hypervisor 保留对少数
控制寄存器、配置、doorbell、调度和特权操作的调解，同时让高频数据提交
尽量直接到硬件。

```text
Guest vendor driver
  |-- control/config access -> Hypervisor mediation
  `-- performance-critical data -> translated/direct hardware path
```

典型目标：

- 比 API remoting 保留更多原生驱动语义；
- 比整设备直通更容易多租；
- 用 trap 控制面换取可调度、可隔离的数据面。

### 7.2 MPT 不是 Linux 子系统

本文按 GPU 架构语境把 MPT 解释为 **Mediated Pass-Through**。它不是类似
VFIO、mdev、SR-IOV 那样的单一内核框架。

一个 MPT 产品可能：

- 用 mdev 做创建和管理；
- 用自定义 PCI capability/MMIO trap；
- 用 PF/VF command queue；
- 用 hypervisor/vendor module 做 privilege mediation；
- 用 IOMMU 和 GPU page table 做 DMA/VM 隔离。

判断 MPT 必须看具体实现，不能只看产品是否自称 “mediated”。

### 7.3 常见设计张力

MPT 的核心不是“完全拦截”或“完全直通”，而是决定哪些操作必须在 Host：

- GPU function reset 和调度；
- 全局时钟/电源管理；
- 特权 firmware command；
- VM 地址空间切换；
- error handling；
- 迁移与 snapshot；
- 安全敏感性能计数器。

这些控制面若拦截不完整，会造成 Guest 破坏共享状态；数据面若拦截过多，则
退化成 driver/UAPI remoting。

## 8. NVIDIA MIG

### 8.1 资源模型

NVIDIA
[MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/latest/)
将 MIG 描述为把受支持 GPU 划分为多个隔离 instance，每个 instance 有独立
计算和显存资源。

概念上有 GPU instance 和 compute instance，实际组合 profile 依 GPU 世代。
MIG 提供：

- 空间资源切分；
- 专用 memory slice；
- 一定的故障/性能隔离；
- 多租户下的确定性；
- 少于整 GPU 的碎片使用。

### 8.2 MIG 不等于完整 VM 虚拟化

把 MIG instance 放进 VM 还需要：

- 平台/Hypervisor 支持；
- 一个 device assignment 接口，可能是 VFIO、mdev 或 vendor driver；
- Guest driver 和 CUDA/ROCm 可见设备语义；
- reset、错误和迁移策略；
- 对 Graphics/Video 等能力的 profile 支持。

MIG 是 GPU 内部分区机制。它可与 VFIO 组合，但 VFIO 不创造 MIG partition。

### 8.3 使用边界

- partition 形状受硬件、profile 和维护模式限制；
- 可能需要停止当前 workload 来重配；
- fragment 可能限制最大单任务；
- MIG 主要针对支持的数据中心 GPU，消费卡通常不支持；
- 图形、视频、CUDA 和 MIG profile 能力并非任一 GPU 上完全相同。

## 9. NVIDIA vGPU 与厂商 vGPU

### 9.1 “vGPU”是产品层

NVIDIA
[vGPU 文档](https://docs.nvidia.com/grid/latest/grid-vgpu-user-guide/index.html)
描述的是 Hypervisor 上的虚拟 GPU 软件栈，包括：

- Host Virtual GPU Manager；
- Guest graphics driver；
- Hypervisor 集成；
- VM 创建/调度；
- migration 与 support matrix；
- licence 管理。

具体硬件世代可能使用不同底层机制，包括 mediated pass-through、SR-IOV
或厂商私有调度。不能把一个版本 vGPU 的底层实现当作所有 vGPU 的定义。

### 9.2 vGPU 与 MIG 不是同义词

| 问题 | MIG | NVIDIA vGPU |
| --- | --- | --- |
| 本质 | GPU 内硬件 partition | 厂商虚拟 GPU 产品/软件栈 |
| Guest 看到 | 取决于如何分配 partition | 厂商虚拟 GPU 设备/profile |
| Guest 驱动 | 厂商 driver，具体设备语义 | vGPU Guest driver |
| 多租方式 | 空间实例 | 厂商 design，可为 MPT/SR-IOV 等 |
| Licence | NVIDIA 不要求 vGPU licence 才能用 MIG | 通常有 vGPU 产品/licence |
| VM 支持 | 需平台额外支持 | 产品本身面向 VM |

vGPU 也可以选择基于 MIG 的 profile 或使用其他 partition 方式，实际组合以
产品版本和 GPU 的 support matrix 为准。

AMD MxGPU 是另一个厂商 vGPU 类方案，常与 AMD SR-IOV 硬件能力关联。具体
功能仍要按 GPU、PF/VF driver 和 Hypervisor 版本核对。

## 10. 方案关系图

```text
                         Guest ABI
                             |
        +--------------------+---------------------+
        |                                          |
  virtual device                              physical-like
        |                                          |
  API remoting / native-UAPI remoting         PCI function/partition
        |                                          |
  VirGL, Venus, GFXStream                    SR-IOV VF
  virtio native context                      VFIO vfio-pci
        |                                    MIG instance + assignment
        |                                    vendor vGPU/MPT
        |
  QEMU/virtio transport
        |
  host renderer/driver

Management/exposure mechanisms: VFIO, IOMMU, mdev, vendor manager
Scheduling/isolation mechanisms: time slice, hardware context, VF, MIG, MPT
```

机制可以叠用：

```text
SR-IOV creates VF -> VFIO exposes VF
Parent registers mdev -> mdev exposes vendor device -> VFIO assigns
Hardware partition (MIG) -> platform exposes partition -> VM
vGPU product -> MPT or SR-IOV -> VM
```

## 11. 横向比较

以下是架构判断，不是固定产品 benchmark：

| 方案 | Guest 看到的 ABI | 人机命令边界 | 典型多租方式 | 迁移 | 主要优势 | 主要代价 |
| --- | --- | --- | --- | --- | --- | --- |
| virtio 2D | virtio-gpu | QEMU 2D command | Host context/share | 较成熟但仍看设备配置 | 简单、可移植 | 3D 有限 |
| VirGL | OpenGL remoting | Host GL | Host renderer context | 当前 QEMU 配置有 blocker | 开源、跨厂商 | 协议/解码/呈现复杂 |
| Venus | Vulkan remoting | Host Vulkan | Host Vulkan context | 状态和协议复杂 | Vulkan 语义完整度高 | feature/版本/WSI |
| GFXStream | 多 API streaming | Host backend | Backend context | Rutabaga 当前 blocker | Android 生态 | 系统大、耦合高 |
| native DRM | driver UAPI-style | Host native driver | Host native contexts | 依 backend | 少高层转换 | Guest/Host ABI 强耦合 |
| VFIO whole GPU | real PCI GPU | Guest vendor driver | 一 VM 独占 | 依 VFIO migration | 行为最接近原生 | 密度/reset/运维 |
| SR-IOV VF | real PCI VF | Guest driver + PF hardware | one VM/VF | 依 PF/VF/platform | 高密度、硬件参与 | 硬件/配额/版本 |
| mdev | vendor-defined | vendor driver | vendor implementation | mdev/vendor UAPI | 无 SR-IOV 也可多实例 | 隔离完全取决于实现 |
| MPT | vendor-like guest driver | Host control mediation + fast path | vendor design | vendor design | 平衡原生与多租 | 复杂、实现差异大 |
| MIG | partition-like | Host/platform assignment | spatial partition | 依平台/profile | 资源与性能隔离 | 形状和图形能力限制 |
| NVIDIA vGPU | vendor virtual GPU | vendor product stack | vendor scheduler | product-specific | VM 运维和完整栈 | licence/support matrix |

## 12. 性能模型

跨层计时、trace correlation、基准设计和瓶颈判定树见
[GPU 虚拟化性能分析与可观测性](../ai-gpu/gpu-performance-observability.md)。

API remoting：

```text
T = T_guest_api
  + T_encode
  + T_virtio
  + T_qemu
  + T_decode
  + T_host_api
  + T_sync
  + T_gpu
  + T_present
```

VFIO：

```text
T = T_guest_driver
  + T_dma_map_and_irq
  + T_hardware_scheduling
  + T_gpu
  + T_present
```

MPT/SR-IOV：

```text
T = T_guest_driver
  + T_control_mediation_or_PF_command
  + T_translation
  + T_hardware_scheduling
  + T_gpu
  + T_present
```

这些公式只帮助建立 profiling 假设。GPU-bound 大型矩阵乘、路径追踪或视频
编码可能让虚拟化协议成本占比很小；大量小 draw、频繁 fence、频繁 map/unmap
会放大 CPU 和往返成本。

必须分别测：

- 单租户吞吐；
- 两/多租户 P50/P99 干扰；
- submit 延迟和 fence 完成；
- 内存拷贝/映射；
- WSI/present；
- reset 和恢复；
- 持续高负载下的 thermal/power fairness。

## 13. 安全模型

跨路径威胁模型、信任边界、数据残留、侧信道和评审检查表见
[GPU 虚拟化的隔离、安全与故障域](../ai-gpu/gpu-virtualization-security.md)。

### 13.1 API remoting

需要防范：

- malicious Guest 构造非法命令长度、offset、handle；
- Host renderer 的 parser 内存安全；
- shader/pipeline compiler attack surface；
- renderer 泄漏 Host 文件、socket、GPU 设备；
- cross-context resource alias；
- DoS 和资源配额。

优点是可把 Host renderer 放独立进程/沙箱；但这需要部署策略，不是协议自动
保证。

### 13.2 VFIO/SR-IOV/MIG

需要防范：

- Guest driver 通过 DMA 越过允许 GPA；
- 设备/firmware 共享状态破坏；
- FLR/reset 后残留 DMA；
- PCIe ACS/bridge backdoor；
- GPU engine 时间片侧信道；
- MIG partition 间未预期信息泄漏；
- error containment 和 DoS。

IOMMU 很重要，但不能证明 GPU firmware、command processor 和硬件 reset
都安全。

### 13.3 判定原则

“Host renderer 在进程外”不等于安全隔离。
“有 SR-IOV VF”不等于所有故障域隔离。
“有 MIG”不等于自动具备 VM ABI。
“VFIO”不等于安全认证。

安全检查最终必须落到 threat model、实现、配置和硬件验证。

## 14. 生命周期与迁移

reset、suspend/resume、snapshot、process checkpoint、hotplug 和 live
migration 的完整状态分类见
[GPU VM 生命周期、热插拔、Checkpoint 与迁移](../ai-gpu/vm-lifecycle-migration.md)。

### 14.1 API remoting

可迁移性取决于 Guest-visible 状态和 Host renderer state：

- context、queue、resource、descriptor；
- shader/pipeline cache；
- fence/timeline；
- Host GPU object 是否可重建；
- vendor driver 版本是否一致；
- display/WSI 转移。

当前仓库中 QEMU VirGL 和 Rutabaga 都明确注册了 migration blocker。不能因为
命令是“远程的”就认为 VM 可以透明迁移。

### 14.2 VFIO

需要 VFIO migration UAPI 和设备驱动实现：

- compatible migration version；
- device state size；
- STOP_COPY/RUNNING 状态机；
- dirty tracking；
- VRAM/system memory transfer；
- destination compatibility；
- error recovery。

很多 GPU 仅支持 reset 或 suspend/resume，不支持 live migration。

### 14.3 SR-IOV/vGPU/MIG

需要同时考虑：

- PF manager 和 VF 状态；
- GPU VM/context/pipeline state；
- VRAM；
- guest driver 是否支持目标 profile；
- host、firmware 和 hypervisor 版本；
- partition 重组；
- vendor licence/feature 是否随 VM 迁移。

GPU 产品的“迁移支持”必须按 source/destination hardware、driver 和
hypervisor 组合验证，而不是只看一个能力位。

## 15. 场景选择

从需求 gate 到候选架构、评分和 rollout 的完整流程见
[GPU 虚拟化架构选型与评审 Playbook](../ai-gpu/gpu-virtualization-design-playbook.md)。

### 15.1 跨厂商 Linux 桌面、开发、CI

scanout、cursor、dma-buf WSI、compositor 和 VDI 路径见
[GPU 显示、Compositor、WSI 与 VDI](../ai-gpu/display-wsi-virtualization.md)。

优先级：

1. virtio-gpu VirGL/Venus；
2. llvmpipe 作为正确性基线；
3. 有硬件条件时增加 VFIO/SR-IOV 对照。

核心目标是可移植性和可观测性，不是最高性能。

### 15.2 Android/Cuttlefish

优先研究：

- GFXStream；
- Rutabaga；
- GLES/Vulkan/compositor 组合；
- dma-buf/cross-domain WSI。

### 15.3 单 VM 高性能原生图形

候选：

- VFIO whole GPU；
- vendor vGPU 独占 profile；
- SR-IOV 单 VF。

先验证 Host 是否可失去该 GPU、IOMMU group、reset、显示和电源。

### 15.4 多租户 cloud

候选：

- SR-IOV VF；
- MIG + platform assignment；
- vendor vGPU；
- API remoting（对隔离/feature 要求可接受时）。

必须测租户间干扰和故障，不仅测单租吞吐。

### 15.5 引入 GPU 云原生平台

容器、Kubernetes device plugin/DRA/CDI、MIG/MPS/time slicing 与隔离层的完整
对照见
[云原生 AI 中的 GPU 共享与虚拟化](../ai-gpu/cloud-native-gpu-sharing.md)。

先确认 K8s/容器不是 VM：

- NVIDIA time-slicing 是同一 GPU 的软件时间共享；
- MIG 是硬件 partition；
- vGPU 是 Hypervisor VM 产品；
- DRA/device plugin 只是调度/暴露层。

把容器编排 `vGPU` 标签当成 VM passthrough 会产生错误安全结论。

## 16. 常见错误结论

### “Guest 显示 NVIDIA，所以是 VFIO”

错。Venus 和 GFXStream 可把 Host GPU 属性转发给 Guest。要看 PCI ID、
capset、QEMU device、Host renderer 和 IOMMU group。

### “virtio-gpu native DRM 就是直通”

错。它仍经过 virtio-gpu、QEMU 和 Host native driver；Guest 没有物理 BAR。

### “mdev 就是 SR-IOV 的软件版本”

不准确。mdev 是没有现成 SR-IOV 时的 mediated device 框架，数据隔离由厂商
实现决定；它不是 PCIe VF 标准。

### “MPT 是某个固定驱动”

错。MPT 是 mediated pass-through 架构模式，可由多个机制实现。

### “MIG 可以替代 vGPU”

错。MIG 是 partition，vGPU 是包含 VM 管理、Guest driver、调度、迁移和许可的
产品栈。二者可组合，不在同一抽象层。

### “SR-IOV 每 VM 都是完全独立的 GPU”

错。VF 的 engine、VRAM、视频能力、调度和故障隔离依赖硬件/profile/PF。

### “VFIO 一定最快”

不一定。GPU-bound workload 可能都受 GPU 限制；VFIO 有 IOMMU、IRQ、reset、
Host 管理成本，且不总是可用。

### “API remoting 可以自然 live migration”

错。Guest visible state、Host renderer state、version、WSI 和 migration
blocker 都可能阻止迁移。

## 17. 方案评审模板

对任何一个新产品或 PR 填下面的表：

```text
Product/feature:
Version/date:

Guest PCI/device ABI:
Guest kernel driver:
Guest userspace library:
Control-plane intercept:
Data-plane transport:
Host execution owner:
Memory objects and ownership:
DMA/IOMMU model:
GPU VA/cache/coherence model:
Completion/fence model:
Display/compositor/scanout:
Multi-tenant scheduling:
Hardware isolation claim:
Software isolation claim:
Reset/error containment:
Migration/snapshot:
Supported hardware/software matrix:
Resource quota/DoS controls:
Observable trace/metrics:
Independent tests:
Known blockers:
```

如果一张表中有“厂商实现”或“依平台”项，就不能用一句方案名替代技术结论。

## 18. Virtio-GPU 的特殊价值

virtio-gpu 不一定是所有生产场景的最优方案，但它是极好的学习平台：

- 能观察 Guest DRM、virtio、QEMU 和 renderer 的完整链路；
- 可用一张 Host GPU 探索多 API remoting；
- 能直接比较 OpenGL、Vulkan、多 API streaming 和 native UAPI；
- failure 通常能定位到明确的协议/资源/fence 层；
- 是理解 VFIO/SR-IOV/MIG 差异的软件基线。

从 virtio-gpu 出发，后续学习顺序应是：

```text
API remoting
  -> native driver UAPI remoting
  -> VFIO whole-device assignment
  -> SR-IOV PF/VF
  -> mdev/MPT vendor mediation
  -> MIG hardware partition
  -> vendor vGPU product stack
```

这不是性能阶梯，而是抽象边界逐层向硬件移动的路线。

Intel 的软件 mediated vGPU 历史实现见
[Intel GVT-g Mediated vGPU 源码跟踪](../ai-gpu/intel-gvt-g-source-trace.md)；
现代 Xe 硬件 VF 路径见
[Intel Xe GPU 虚拟化与 SR-IOV](../ai-gpu/intel-xe-sriov.md)。

Windows/Hyper-V 的 DDA、GPU-P、WDDM 与 WSL2 差异见
[Windows/Hyper-V GPU 虚拟化](../ai-gpu/windows-hyperv-gpu-virtualization.md)。

## 19. 检查题

1. 为什么 VFIO 不是一套 GPU command protocol？
2. SR-IOV VF 和 mdev 的 PCIe/总线身份有什么差别？
3. mdev 为什么不能自动保证硬件隔离？
4. MPT 的控制面和数据面分别可能拦截什么？
5. MIG 为什么不是完整 VM ABI？
6. NVIDIA vGPU 为什么可能使用不同底层资源切分机制？
7. virtio-gpu native DRM context 缺少哪些 VFIO 特性？
8. API remoting 和 VFIO 的主要攻击面各是什么？
9. 为什么 “GPU-bound 测试接近原生” 不能代表所有 workload？
10. 判断 live migration 时必须核对哪些 Guest/Host state？
11. 在 K8s 中，time-slicing、MIG 和 vGPU 各属于什么层？
12. 设计高密度多租 GPU 云时，如何同时评估吞吐、P99 干扰和安全？
