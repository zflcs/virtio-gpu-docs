# VFIO、IOMMUFD、SR-IOV 与 mdev 通用机制源码跟踪

## 1. 范围与证据边界

对应宏观文档：
[GPU 虚拟化：硬件、软件与 API 分层](../virtio-gpu/platform-comparison.md)。

本文跟踪 Linux 设备虚拟化的通用机制，再把它们映射到 GPU 和 AI compute。
重点不是重复 GPU 厂商实现，而是回答：

- VFIO 究竟拦截什么，不拦截什么；
- legacy VFIO group/container 与 device cdev 的窃取和生命周期有何不同；
- IOMMUFD 的 IOAS、HWPT_PAGING、HWPT_NESTED、vIOMMU 分别表示什么；
- PCI SR-IOV 如何创建 VF，VF 为什么仍不等于独立 GPU；
- mdev 如何生成一个可交给 VFIO 的厂商定义设备；
- 这些机制如何变成 Guest CUDA/ROCm 可见的 PCI function、BAR、MSI-X、
  DMA 和 GPU queue 边界。

源码基线：

```text
Linux 6b4b543fa18757297ac756a6fe6e64f506b7844a
```

本文不运行 VM、绑定设备、创建 VF、运行 GPU workload 或迁移实验。源码中
存在接口，不等于某块 GPU、firmware、hypervisor 和 Guest driver 的组合已经
支持该能力。

证据等级：

| 结论类型 | 证据 |
| --- | --- |
| Linux 结构、ioctl、所有权检查 | 固定 commit 源码 |
| VFIO/IOMMUFD 设计方向 | Linux 文档加源码 |
| 某 GPU 的 VF 数量、MIG profile、vGPU profile | GPU/厂商文档 |
| 某组合可安全 reset、P2P 或 live migration | 固定硬件与软件版本下的运行证据 |

## 2. 一张图建立正确模型

```text
Host physical device
  -> PCI function / VF / MIG/vGPU/vendor virtual device
  -> VFIO device
       +-> BAR / config / interrupt / reset
       +-> DMA ownership
            -> IOMMUFD IOAS/HWPT
            -> IOMMU translation
  -> Guest vendor driver
       -> CUDA/ROCm or graphics UAPI
       -> GPU queues and embedded command buffers
```

必须分开的五层：

| 层 | 典型对象 | 主要问题 |
| --- | --- | --- |
| PCIe/function | PF、VF、RID、BAR、MSI-X | Guest 看见什么设备 |
| DMA ownership | VFIO group 或 IOMMUFD device | 谁独占设备 DMA 的控制权 |
| I/O address space | container、IOAS、HWPT | GPA/VA 如何映射成 IOVA |
| Device programming | config、BAR、IRQ、ioeventfd、reset | Guest 如何驱动设备 |
| GPU execution | queue、doorbell、context、fence、VRAM | 谁调度 compute、谁拥有显存 |

VFIO 主要解决前三层和第四层的通用暴露问题。它不会解释 Vendor GPU 的
draw 或 kernel command，也不会自动提供 GPU 内部的 VRAM/queue 隔离。

## 3. 四条生命周期

### 3.1 Legacy VFIO group/container

```text
bind physical device to vfio-pci
  -> kernel creates IOMMU group and device fd
  -> userspace opens /dev/vfio/vfio
  -> opens /dev/vfio/<group>
  -> VFIO_GROUP_SET_CONTAINER
  -> VFIO_SET_IOMMU
  -> VFIO_IOMMU_MAP_DMA
  -> VFIO_GROUP_GET_DEVICE_FD
  -> config/BAR/IRQ/reset programming
```

container 是共享 page table 的逻辑容器；group 是可归属的隔离单元。一个
container 可以包含多个 group，但 group 仍是 DMA ownership 的最小安全粒度。

### 3.2 Device cdev + IOMMUFD

```text
bind physical device to vfio-pci
  -> open /dev/vfio/devices/vfioX
  -> open /dev/iommu
  -> VFIO_DEVICE_BIND_IOMMUFD
  -> IOMMU_IOAS_ALLOC
  -> IOMMU_IOAS_MAP
  -> VFIO_DEVICE_ATTACH_IOMMUFD_PT
  -> config/BAR/IRQ/reset programming
```

打开 cdev 本身不授予设备访问权。`BIND_IOMMUFD` 成功后才建立 device ownership
和完整的 UAPI access。modern path 用 IOMMUFD object 替代 legacy container
和 VFIO IOMMU driver 的大量逻辑。

### 3.3 SR-IOV VF

```text
PF driver exposes SR-IOV capability
  -> host writes sriov_numvfs
  -> PCI core enables VF Enable + Memory Space Enable
  -> VFs become hot-pluggable PCI functions
  -> bind selected VF to vfio-pci
  -> assign that VF through group/container or cdev/IOMMUFD
```

Guest 使用真实 VF driver；Host PF driver、firmware 和调度器仍管理 VF
provisioning、资源配额以及可能的时间片调度。

### 3.4 mdev

```text
parent GPU driver registers mdev parent and types
  -> userspace writes UUID to supported-type/create
  -> mdev core creates mdev_device
  -> vendor VFIO/mdev driver binds it
  -> mdev joins an emulated/mediated DMA ownership model
  -> userspace obtains VFIO device access
```

mdev 统一了设备管理和 VFIO 接入，不规定 vendor 如何隔离 VRAM、engine 或
firmware。NVIDIA vGPU 是否采用 mdev、SR-IOV 或私有 MPT，必须看具体产品。

## 4. VFIO 注册与两种访问路径

### 4.1 从 kernel driver 注册 VFIO device

GPU bus driver 调用
[`vfio_register_group_dev()`](../../linux/drivers/vfio/vfio_main.c:402)
把 `struct vfio_device` 注册到 VFIO core。注册路径还要求传统 IOMMU-backed
device 满足 cache-coherent DMA：

```c
if (type == VFIO_IOMMU && !vfio_device_is_noiommu(device) &&
    !device_iommu_capable(device->dev, IOMMU_CAP_CACHE_COHERENCY))
    return -EINVAL;
```

这是因为 VFIO 不提供让用户恢复 cache coherence 的协议。对 GPU 来说，
Host 分配的内存、Guest backing 页和 GPU cache 之间是否一致，必须在更下层的
driver/firmware 中解决。

`vfio_register_emulated_iommu_dev()` 注册没有直接 DMA 的虚拟设备，mdev 和
软件模拟设备常走这一类。它不代表设备没有数据路径，只表示 DMA 需要经过
可审计的访问接口。

### 4.2 Legacy group 是 ownership 单元

group 文件支持的主要 ioctl：

- `VFIO_GROUP_GET_STATUS`
- `VFIO_GROUP_SET_CONTAINER`
- `VFIO_GROUP_UNSET_CONTAINER`
- `VFIO_GROUP_GET_DEVICE_FD`

[`vfio_group_fops_unl_ioctl()`](../../linux/drivers/vfio/group.c:364)
负责分发。`SET_CONTAINER` 后，container path 中的
[`iommu_group_claim_dma_owner()`](../../linux/drivers/vfio/container.c:437)
声明整个 group 的 DMA ownership。

因此即使 Guest 只要一个 GPU function，只要它和其他设备共享不可隔离的
IOMMU group，其他 function/bridge 也必须可被安全归属。

### 4.3 Device cdev 的安全门

用户先打开
[`/dev/vfio/devices/vfioX`](../../linux/Documentation/driver-api/vfio.rst:269)。
[`vfio_df_ioctl_bind_iommufd()`](../../linux/drivers/vfio/device_cdev.c:92)
会：

1. 校验 `argsz`、flags 和 iommufd fd；
2. 阻止同一个 group 同时通过 legacy 和 cdev 路径打开；
3. 限制一个 device 同时只有一个有效 cdev opener；
4. 从用户 fd 取得 `iommufd_ctx`；
5. 调用 device-specific `bind_iommufd`；
6. 建立 KVM/device state；
7. 最后用 release-store 发布 `access_granted`。

其他 read/write/mmap/ioctl 使用 acquire-load 检查这个标志。源码顺序表达的是
安全边界：打开设备文件只是取得后续绑定能力，不是取得未映射的物理 DMA 能力。

## 5. IOMMUFD 的对象模型

### 5.1 IOAS

[`iommufd_ioas_map()`](../../linux/drivers/iommu/iommufd/ioas.c:244)把用户 VA
区间映射到选定 IOVA。核心步骤是：

1. 校验 map flags 至少包含 readable/writable；
2. 取得 `iommufd_ioas`；
3. 固定用户页；
4. 在 `io_pagetable` 中建立 IOVA -> PFN；
5. 同步所有关联的 paging HWPT；
6. 把实际 IOVA 返回用户。

源码文档明确说明：多个 IOAS 可共享同一底层 `iopt_pages`，从而避免同一批
页被重复 pin 和记账。这是 AI 大块显存/主机内存映射的重要设计，但共享 page
不代表不同租户可直接共享。

### 5.2 Device bind 与 attach

[`iommufd_device_bind()`](../../linux/drivers/iommu/iommufd/device.c:310)
取得物理 device 的 DMA ownership，并返回 device ID。
[`iommufd_device_attach()`](../../linux/drivers/iommu/iommufd/device.c:1081)
再把 device/PASID 连接到 IOAS 或显式 HWPT。

概念关系：

```text
object ID namespace (iommufd fd)
  IOAS: IOVA -> PFN map
    -> HWPT_PAGING: actual stage-2 I/O page table
      -> DEVICE / PASID attachment
```

VFIO device cdev 通过
`VFIO_DEVICE_ATTACH_IOMMUFD_PT` 调 device-specific `attach_ioas`，再把生成的
`pt_id` 返回给 VMM。

### 5.3 Nested translation

两级地址转换：

```text
device IOVA
  -> Guest-owned stage-1 page table: IOVA/GVA -> GPA
  -> Host-managed stage-2 page table: GPA -> HPA
  -> Host physical memory / peer memory
```

IOMMUFD `HWPT_NESTED` 表示由 Guest/userspace 管理的 stage-1；
`HWPT_PAGING` 可作为写入 GPA -> HPA 映射的 nesting parent。
这使 vIOMMU、PASID、ATS/PRI 和 confidential VM 的 DMA 访问可以保留更完整的
Guest 地址语义。

硬件支持、Guest vIOMMU 实现、IOMMU interrupt remapping 和 fault delivery
必须同时可用。仅创建一个 nested object 不能证明完整 nVM。

### 5.4 vIOMMU 与 vDEVICE

固定源码 UAPI 还包含：

- `IOMMU_VIOMMU_ALLOC`
- `IOMMU_VDEVICE_ALLOC`
- `IOMMU_VEVENTQ_ALLOC`
- `IOMMU_HW_QUEUE_ALLOC`

Linux 文档把 vIOMMU 定义为物理 IOMMU 的切片，而不是必然与 Guest 里的一个
虚拟 IOMMU 一一对应。vDEVICE 保存该物理设备在 vIOMMU 中的 virtual ID
等属性。对 GPU 虚拟化而言，这些对象可能成为 Guest PASID/RID、fault queue
和 invalidation queue 的硬件加速边界。

这部分是当前 API 演进方向。支持矩阵必须以固定内核、IOMMU 和 VMM 版本为准。

### 5.5 Legacy compatibility

固定 Linux 文档仍说明 legacy VFIO group/container 可通过 IOMMUFD
compatibility layer 实现，但建议新用户逐步迁移到 cdev + native IOMMUFD。
“计划 deprecated”不等于当前内核已经删除 group path。评审部署时必须看实际
内核配置和应用。

## 6. VFIO PCI 做了什么

### 6.1 Device open

[`vfio_pci_open_device()`](../../linux/drivers/vfio/pci/vfio_pci.c:103)
调用 `vfio_pci_core_enable()`，完成：

- PCI device enable；
- reset capability 检测；
- PCI config 和 save state；
- INTx 兼容处理；
- device-specific region 初始化；
- MSI-X table 位置和 BAR 检测；
- VGA region 判定；
- BAR resource 映射准备。

关闭时 [`vfio_pci_core_disable()`](../../linux/drivers/vfio/pci/vfio_pci_core.c:670)
先恢复电源状态，清除 bus master，停止 interrupt/ioeventfd 路径，再释放 BAR
和配置资源。顺序错误可能让仍能 DMA 的设备指向已经释放的内存。

### 6.2 Region、BAR 和 mmap

[`vfio_pci_core_mmap()`](../../linux/drivers/vfio/pci/vfio_pci_core.c:1803)
按 region index 处理 config、ROM、BAR 或 vendor region。主要检查：

- mapping 必须 shared；
- BAR 必须允许 mmap；
- 请求范围不能跨 BAR；
- BAR resource 仍属于该设备；
- runtime power 和 Memory Space Enable 状态允许访问。

[`vfio_pci_vmf_insert_pfn()`](../../linux/drivers/vfio/pci/vfio_pci_core.c:1743)
在 runtime suspend 或 Memory Space 被禁用时返回 `VM_FAULT_SIGBUS`。
Guest 看到 BAR 存在，不代表 Host runtime PM 或 reset 期间还能安全访问。

### 6.3 Config space

[`vfio_pci_rw()`](../../linux/drivers/vfio/pci/vfio_pci_core.c:1623)
把读写分到 config、ROM、BAR、VGA 和 vendor region。config space 不是任意
raw passthrough，因为很多位由 VFIO 的权限和虚拟化规则处理。

GPU 评审要特别检查：

- SR-IOV capability 和 VF Enable 暴露给谁；
- Bus Master Enable 能否在错误路径被清除；
- BAR resizing/assignment 是否被允许；
- MSI-X enable 和 PBA/table 位置；
- FLR 能力与 `NoSoftRst` 行为；
- AER、PM、VGA、ROM 的可见性。

### 6.4 中断

VFIO 通过 `VFIO_DEVICE_GET_IRQ_INFO` 描述 IRQ，通过
`VFIO_DEVICE_SET_IRQS` 建 eventfd/trigger 等映射。`VFIO_DEVICE_IOEVENTFD`
允许某些 MMIO doorbell 写直接触发 eventfd，减少每次写 doorbell 的 VMM
完整处理开销。

IOMMU interrupt remapping 不完整时，攻击者可能伪造 interrupt identity。
TYPE1 attach path 在没有 isolated MSI 且没有显式 unsafe 参数时拒绝设备。

### 6.5 Reset 与 hot reset

VFIO 提供：

- `VFIO_DEVICE_RESET`：单个 device reset；
- `VFIO_DEVICE_GET_PCI_HOT_RESET_INFO`；
- `VFIO_DEVICE_PCI_HOT_RESET`。

hot reset 影响共享 slot/bridge 的 function，可能要求同一 reset domain 中其他
设备也具有兼容 ownership。GPU reset 不是“One VM FLR 就完全隔离”的同义词；
VRAM 中尚未完成的 DMA、firmware hang 和 Host 上其他 function 都必须纳入
failure domain。

### 6.6 VFIO device feature

[`vfio_ioctl_device_feature()`](../../linux/drivers/vfio/vfio_main.c:1227)
通过统一 feature namespace 暴露可选能力。与 GPU/AI 相关的重要项包括：

- migration；
- migration device state；
- DMA logging；
- low-power entry/exit；
- bus-master set/clear；
- DMA-BUF；
- PCI VF token。

“feature 可探测”只是兼容性入口；只有 `GET` 成功且 device-specific callback
实现完整语义，能力才真正可用。

## 7. IOMMU 地址空间与 DMA

### 7.1 Legacy TYPE1 attach

[`vfio_iommu_type1_attach_group()`](../../linux/drivers/vfio/vfio_iommu_type1.c:2240)
大致执行：

1. 查重并检查当前 IOVA table 是否可修改；
2. 为 group 分配 IOMMU domain；
3. `iommu_attach_group()`；
4. 检查 domain aperture 和 IOVA 冲突；
5. 获取 reserved regions；
6. 从 IOVA range 中排除设备保留区；
7. 处理 software MSI；
8. 检查 interrupt remapping/isolation；
9. 成功后才把 group 纳入 container。

reserved region 可能来自 IOMMU、PCI bridge 或平台限制。只检查“map ioctl
返回成功”不能证明 Guest 整个 GPA range 都能直通。

### 7.2 Map 与 pin

`VFIO_IOMMU_MAP_DMA` 通过
[`vfio_dma_do_map()`](../../linux/drivers/vfio/vfio_iommu_type1.c:1684)
建立 IOVA 映射，并 pin 用户 backing page。
`VFIO_IOMMU_UNMAP_DMA` 在获准后解除映射和 pin。

对 AI workload：

- 模型权重/KV cache 通常长期占用数 GB 到数百 GB；
- 重复 pin/unpin 会放大 munmap、page migration 和 TLB 成本；
- dirty bitmap 与 migration 必须覆盖设备写入；
- P2P mapping 和普通 system-memory mapping 的 ownership 不同；
- Guest page 被回收/migration 后，Host 旧 IOVA 和 GPU TLB 必须失效。

### 7.3 IOMMUFD map

`IOMMU_IOAS_MAP`、`IOMMU_IOAS_MAP_FILE` 和 `IOMMU_IOAS_COPY` 分别服务：

- 用户 VA -> IOVA；
- 文件/dma-buf backing -> IOVA；
- 从一个 IOAS 复制现有 page mapping。

`COPY` 复制 page reference 和映射，不复制 GPU framebuffer 内容。资源内容是否
一致仍由 dma-buf、copy engine 或 coherency 协议保证。

## 8. PCI SR-IOV 源码路径

### 8.1 能力发现

[`sriov_init()`](../../linux/drivers/pci/iov.c:771)读取 SR-IOV Extended
Capability，建立 `struct pci_sriov`，记录：

- total/initial VF 数；
- VF offset、stride、device ID；
- VF BAR 大小；
- bus range；
- Migration interrupt/status 支持；
- PF/VF dependency link。

### 8.2 开启 VF

[`sriov_enable()`](../../linux/drivers/pci/iov.c:624)检查后执行：

1. 已有 VF 时拒绝重复 enable；
2. 校验 initial/total/requested VF 数；
3. 检查每个 VF BAR 所需 MMIO 资源；
4. 检查 VF bus number 是否超出上游 bus range；
5. `pci_enable_resources()`；
6. 设置 PF SR-IOV Control 的 VFE 和 MSE；
7. 等待 hardware 状态稳定；
8. 逐个创建 VF PCI function；
9. 发布 uevent。

[`pci_iov_add_virtfn()`](../../linux/drivers/pci/iov.c:321)扫描 VF config，
把每个 VF BAR 切成 PF IOV BAR 的子 range，建立 sysfs `virtfnN` 链接，并作为
PCI device 加入总线。

### 8.3 关闭 VF

`sriov_disable()` 先移除 VF device，再清除 VFE/MSE、等待、回收 bus/MMIO
资源并发布状态变化。一个失败的 enable 也必须回滚配置和 VF device。

### 8.4 SR-IOV 提供了什么

PCI core 保证 VF 具有独立 Routing ID、config space 和 MMIO range。
GPU firmware/PF driver 进一步决定：

- VF 是否拥有独立 scheduler queue；
- VRAM/GTT quota；
- VF 之间是否能 P2P；
- 错误是否 FLR 隔离；
- VF 是否可暂停、保存和迁移。

因此“VF 出现在 `lspci`”只证明 PCI function 被创建，不证明 AI workload
拥有独立 GPU 或安全隔离。

## 9. mdev 源码路径

### 9.1 Parent 和 type 注册

[`mdev_register_parent()`](../../linux/drivers/vfio/mdev/mdev_core.c:59)
保存 parent device、mdev driver、类型数组和可用 instance 数，创建
`mdev_supported_types` sysfs 层级。

Type 描述设备 API、名称、description 和剩余 instance。厂商驱动决定每个
type 代表什么资源 profile。

### 9.2 创建 mediated device

用户向 type 的 `create` 写 UUID：

1. [`create_store()`](../../linux/drivers/vfio/mdev/mdev_sysfs.c:61)
   解析固定长度 UUID；
2. `mdev_device_create()` 检查 UUID 重复；
3. 检查 `available_instances`；
4. 建立 mdev bus device 和父设备关系；
5. 绑定注册好的 mdev driver。

mdev core 不解析 GPU command，也不创建 VRAM。vendor driver 的 probe 必须
建立真正设备对象，再注册 VFIO device ops。

### 9.3 mdev 与 DMA

mdev 可以：

- 使用硬件/固件提供 mediated DMA；
- 通过 IOMMUFD access 对 IOAS page 做受控读写；
- 通过 vendor queue/doorbell 执行命令；
- 只模拟控制面，把数据转发给 Host renderer。

因此 mdev 的隔离、性能和 reset 都由具体 backend 决定。mdev framework 只
保证设备创建、sysfs 管理、总线绑定和 VFIO 接入方式统一。

## 10. PASID、ATS/PRI、P2P 与 AI

GPU compute 常需要 per-process 地址空间和 page fault。IOMMUFD/VFIO 源码
允许对支持 device op 的 PASID 单独 attach IOAS，但仍受以下条件约束：

- GPU 是否发送正确 PASID；
- IOMMU 是否支持 PASID 和 nested translation；
- ATS/PRI 是否可用且可信；
- GMMU/SMMU 与 CPU MM/HMM 的 fault 协议；
- VF 是否允许相应 PASID 数量；
- IOMMU group/ACS 是否阻断不期望的 P2P。

AI 训练中的 NCCL/RCCL、IPC 和 peer tensor 会依赖 P2P。IOMMU 保护 DMA 与
允许 P2P 是不同问题；ACS 或 topology 可能让看似可用的 cudaMemcpyPeer/HIP
P2P 退化或失败。

详细地址链和 ACS 见
[PCIe、IOMMU、ATS/PASID 与 GPU P2P](pcie-iommu-ats-p2p.md)。

## 11. VFIO migration 状态机

[`vfio_ioctl_device_feature_migration()`](../../linux/drivers/vfio/vfio_main.c:989)
探测 STOP_COPY、P2P、PRE_COPY 等能力。
`VFIO_DEVICE_FEATURE_MIG_DEVICE_STATE` 实现 opaque migration stream 的有限
状态机：

```text
RUNNING
  -> PRE_COPY*
  -> RUNNING_P2P*
  -> STOP
  -> STOP_COPY
  -> RESUMING
  -> STOP
  -> RUNNING
```

`*` 表示设备可选支持的 P2P/pre-copy 状态。`ERROR` 只能通过 reset 恢复。

关键约束：

- STOP 状态下设备不得继续 DMA、中断或改变外部状态；
- STOP_COPY 期间用户不能继续操作设备 region；
- migration stream 是设备 driver 定义的私有格式；
- 源和目的设备必须满足厂商的兼容矩阵；
- pre-copy 期间 P2P 必须被正确处理。

存在状态机不等于 GPU 可迁移。GPU migration 还要保存/重建 VRAM、GPU page
table、queue、fence、firmware 和多 GPU collective 状态。详见
[GPU VM 生命周期、热插拔、Checkpoint 与迁移](vm-lifecycle-migration.md)。

## 12. GPU 虚拟化方案如何落到这些机制

| 方案 | Guest 设备 | Command 执行者 | DMA/IOMMU | 生命周期边界 |
| --- | --- | --- | --- | --- |
| VFIO 整卡 | 物理 PCI function | Guest vendor KMD/GPU | VFIO + IOMMU | 独占、FLR/reset、可选 migration |
| SR-IOV VF | PCI VF | Guest VF driver + PF/firmware | VFIO + IOMMU | VF、profile、VF reset/migration |
| mdev/vGPU | vendor virtual device | vendor mediated backend | emulated/mediated + IOMMU | vendor manager，非固定 |
| MPT | vendor-visible function/device | Guest driver + hypervisor mediation | 部分直接、部分 trap | vendor-specific |
| MIG | GPU internal GI/CI | Guest/Host vendor stack 使用 partition | 资源 partition，再接 VFIO/vGPU 等 | profile 与 partition reset |
| virtio-gpu | `1af4:1050` virtual device | Guest UMD + Host renderer | virtqueue/mem entry/blob + renderer | QEMU/renderer，当前多为软件生命周期 |

重要区分：

- SR-IOV 是 PCIe function 机制；
- VFIO 是 device ownership、I/O region 和 DMA 暴露机制；
- IOMMUFD 是 userspace 管理 IO page table/device/PASID 的接口；
- mdev 是 vendor-mediated device 管理框架；
- MPT 是 hypervisor mediation 策略；
- MIG 是 GPU 内部资源 partition；
- vGPU 是产品级组合，不是单一内核机制；
- virtio-gpu 是虚拟设备协议和 Host renderer 架构。

## 13. AI workload 的架构含义

### 13.1 CUDA/ROCm 二进制兼容

VFIO/SR-IOV/vGPU 通常让 Guest 加载原生 vendor driver，因此最容易保留 CUDA、
ROCm、CUDA Graph、IPC 和第三方库 ABI。virtio-gpu Venus/GFXStream 保留的是
Vulkan/GFXStream 协议，不自动提供 CUDA/HIP 二进制兼容。

### 13.2 内存密度

VFIO 整卡给一个 VM 时，HBM quota 基本是整卡。SR-IOV/mdev/vGPU 需要 PF
和 firmware 管理 VRAM quota。MIG 通过硬件 fragment 和 profile 分区。
API remoting 则由 Host renderer 为多个 Guest 管理资源。

容量相同不表示故障域相同：共享 HBM 控制器、L2 cache、copy engine 或 power
预算仍可能产生租户干扰。

### 13.3 Queue 和 doorbell

AI compute 的高 submit rate 要求 queue memory、wptr/rptr 和 doorbell 尽量
留在性能路径。VF/MPT 通过 Guest driver 直接或半直接写设备；API remoting
要经过 parser/proxy。deferred decode 这类小 kernel workload 对额外边界更
敏感。

### 13.4 Pinning 与 SVM

VFIO map 会 pin backing page；userptr/SVM 可能依赖 fault 和 migration。
若 Guest/Host 内存模型不能统一，可能出现：

- 长生命周期 pin 阻止回收；
- GPU TLB 保留失效页映射；
- Host migration 后 Guest 看不到新位置；
- 多 GPU peer mapping 与 Host topology 不一致。

### 13.5 Reset 与 checkpoint

VFIO reset 是 device reset UAPI；AI job 恢复还需要 CPU process、GPU VRAM、
KFD/HSA queue、CUDA context、collective communicator 和分布式 scheduler
状态。VM reset 不等于 workload checkpoint。

## 14. 失败矩阵

| 症状 | 最先检查 |
| --- | --- |
| `VFIO_GROUP_SET_CONTAINER` 失败 | group viability、DMA owner、其他 function |
| `VFIO_DEVICE_BIND_IOMMUFD` 返回 `EBUSY` | group/device 是否已有 opener、legacy 与 cdev 是否混用 |
| IOAS map 失败 | 用户页范围、IOVA aperture、reserved region、flag |
| attach 失败 | device op、PASID/nested 能力、IOMMU 驱动 |
| BAR mmap 返回 `SIGBUS` | runtime PM、Memory Space、reset/VF 状态 |
| MSI-X 工作异常 | IRQ remapping、MSI-X table BAR、VF/PF 权限 |
| SR-IOV enable 返回 `ENOMEM` | VF BAR MMIO、bus number、resource reservation |
| mdev create 失败 | type instance 数、UUID、vendor parent 状态 |
| Guest GPU 可见但 CUDA/ROCm 失败 | VF/UUID/topology、driver/firmware 兼容 |
| 单卡正常但 collective 失败 | P2P/ACS、IOMMU topology、NCCL/RCCL peer discovery |
| reset 影响同 slot 设备 | PCI hot reset domain、共享 bridge |
| migration 无法开始 | feature GET、STOP_COPY、BMS、P2P、厂商兼容矩阵 |

## 15. 专家级审查清单

### 15.1 设备与所有权

- Guest 看到 PCI ID、VF、mdev UUID 还是 virtio-gpu？
- IOMMU group 中还有哪些 device/function？
- 谁声明 DMA ownership，如何拒绝第二个 owner？
- no-IOMMU 是否被关闭？
- interrupt remapping 和 isolated MSI 是否成立？

### 15.2 地址空间

- Guest VA/GPA、IOVA、Host VA/HPA、GPU VA、VRAM 如何映射？
- IOAS/HWPT 是自动 domain 还是显式/nested domain？
- 哪些页被 pin，何时 unpin？
- fault、TLB invalidate 和 dirty tracking 谁负责？
- dma-buf import/export 是否保留生命周期和 modifier？

### 15.3 执行与同步

- Guest driver 是否真正控制 GPU queue/doorbell？
- PF/firmware 是否做时间片调度？
- submit、completion 和 present 分别是哪个 fence？
- reset 时哪些 queue/fence 被取消或 signal？

### 15.4 AI 兼容性

- 现有 CUDA/HIP binary 是否无需改写？
- CUDA Graph、IPC、VMM、P2P 和 third-party library 是否支持？
- VRAM quota、NUMA、PCIe/NVLink/XGMI topology 是否与 Guest 视图一致？
- 推理 decode、训练 collective 和 checkpoint 是否分别验收？

### 15.5 生命周期

- hotplug、FLR、PCI hot reset、device reset 的状态机是什么？
- suspend/hibernate/checkpoint/live migration 各支持哪一层？
- source/destination device、firmware 和 driver 版本如何匹配？
- failure 后能否回到 RUNNING，还是必须重建 VM/job？

## 16. 推荐源码阅读顺序

1. `Documentation/driver-api/vfio.rst`：group、container、cdev、IOMMUFD。
2. `drivers/vfio/vfio_main.c`：VFIO device 注册、feature 和 migration。
3. `drivers/vfio/group.c` 与 `container.c`：legacy ownership 路径。
4. `drivers/vfio/device_cdev.c` 与 `iommufd.c`：现代绑定和 attach。
5. `drivers/iommu/iommufd/ioas.c`：IOAS map/unmap/copy。
6. `drivers/iommu/iommufd/device.c`：bind、attach、PASID、access。
7. `drivers/vfio/pci/vfio_pci_core.c`：BAR/config/IRQ/reset。
8. `drivers/vfio/vfio_iommu_type1.c`：legacy DMA mapping。
9. `drivers/pci/iov.c`：SR-IOV enable/disable 和 VF 创建。
10. `drivers/vfio/mdev/`：mdev parent/type/device/sysfs。
11. Intel Xe 和 AMD amdgpu/KFD 的 PF/VF 实现。

GPU 厂商源码专题：

- [Intel Xe GPU 虚拟化与 SR-IOV](intel-xe-sriov.md)
- [AMD GPU SR-IOV 与 MxGPU](amd-gpu-sriov-mxgpu.md)
- [NVIDIA CUDA、MIG 与 vGPU](nvidia-cuda-vgpu.md)
- [PCIe、IOMMU、ATS/PASID 与 GPU P2P](pcie-iommu-ats-p2p.md)

## 17. 检查题

1. 为什么 VFIO 的归属单元是 IOMMU group，而不是单个 device fd？
2. IOMMUFD IOAS 和 HWPT 分别表示逻辑映射与实际硬件页表吗？
3. `VFIO_DEVICE_BIND_IOMMUFD` 为什么必须早于 config/BAR 访问？
4. legacy `MAP_DMA` 与 IOMMUFD `IOAS_MAP` 的共同点和差异是什么？
5. VFIO device cdev 为什么仍受 IOMMU group 约束？
6. SR-IOV enable 时为什么要检查 VF BAR、bus range 和 dependency link？
7. VF 创建成功为什么不能证明每 VF 有独立 GPU engine？
8. mdev framework 能保证厂商 backend 的什么，不能保证什么？
9. PCI hot reset 为什么不等于单个 VF 的安全 reset？
10. VFIO migration feature bit 存在时，为什么仍不能推断某 GPU 可迁移？
11. PASID attach 成功需要哪些 IOMMU、GPU 和 fault 机制共同支持？
12. 为什么 VFIO 整卡保留 CUDA/ROCm 兼容性，却不解决多租密度？
13. virtio-gpu ROCm proposal 与 SR-IOV VF 在 Guest 设备模型和执行所有者
    上分别差在哪里？
14. AI 推理 decode 和训练 collective 分别会放大哪些虚拟化边界？
15. 如何用源码、版本矩阵和运行 trace 分清“接口存在”和“产品可用”？

## 18. 与其他专题的关系

- [GPU 虚拟化平台比较](../virtio-gpu/platform-comparison.md)
- [QEMU VFIO 源码跟踪](qemu-vfio-source-trace.md)
- [GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)
- [GPU 执行模型、队列、调度与故障隔离](gpu-execution-model.md)
- [GPU 虚拟化性能分析与可观测性](gpu-performance-observability.md)
- [GPU VM 生命周期、热插拔、Checkpoint 与迁移](vm-lifecycle-migration.md)
- [AI Runtime、API 与虚拟化兼容性](ai-runtime-api-compatibility.md)
- [云原生 AI 中的 GPU 共享与虚拟化](cloud-native-gpu-sharing.md)
- [GPU 虚拟化架构选型与评审 Playbook](gpu-virtualization-design-playbook.md)
