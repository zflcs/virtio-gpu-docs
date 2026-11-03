# QEMU VFIO 源码跟踪：从 Host fd 到 Guest PCI

## 1. 范围

对应 kernel 侧文档：
[VFIO、IOMMUFD、SR-IOV 与 mdev 通用机制源码跟踪](vfio-iommufd-sriov-source-trace.md)。

本文跟踪 QEMU 如何把 Host VFIO/IOMMUFD 设备变成 Guest 可见的 PCI function：

```text
Host PCI function
  -> /sys/bus/pci/devices/...
  -> /dev/vfio/devices/vfioX or /dev/vfio/<group>
  -> VFIO container backend
  -> QEMU MemoryListener / IOAS
  -> VFIO PCI config / BAR / IRQ / reset
  -> Guest vendor KMD and UMD
  -> GPU queue / VRAM
```

源码基线：

```text
QEMU d73149283205645625bfa9fe029cfb68e4c4a9ae
Linux 6b4b543fa18757297ac756a6fe6e64f506b7844a
```

本文不做设备绑定、VM 启动、VF 创建或迁移实验。QEMU 接受某个 ioctl 或建立
某个 region，只能证明这条软件路径存在；GPU firmware、reset、P2P 和迁移
能力仍需固定产品组合的运行证据。

## 2. 当前 QEMU 的模块边界

固定版本已把过去集中在少量 `vfio*.c` 中的逻辑拆分：

| 源码 | 责任 |
| --- | --- |
| `hw/vfio/device.c` | 通用 VFIODevice、IOMMU backend 选择、device list、IRQ helper |
| `hw/vfio/container.c` | 通用 container 抽象、地址空间、DMA map/unmap、dirty tracking |
| `hw/vfio/container-legacy.c` | `/dev/vfio/<group>` + `/dev/vfio/vfio` backend |
| `hw/vfio/iommufd.c` | `/dev/vfio/devices/vfioX` + `/dev/iommu` backend |
| `hw/vfio/listener.c` | Guest AddressSpace MemoryListener 到 Host IOMMU 的映射 |
| `hw/vfio/region.c` | VFIO region、BAR/MMIO、mmap、sparse mmap、dma-buf |
| `hw/vfio/pci.c` | Guest PCI function、config、BAR、IRQ、quirks、reset |
| `hw/vfio/migration.c` | VFIO opaque migration stream 和 QEMU migration hooks |
| `hw/vfio/display.c` | 直通 GPU 的 display/ramfb 辅助路径 |
| `hw/vfio/igd.c`, `pci-quirks.c` | Intel IGD 与 PCI/vendor workaround |

这一拆分说明：VFIO 在 QEMU 中不是一条固定调用栈，而是“通用 VFIO PCI
设备 + 可替换 IOMMU backend”。

## 3. 两个后端，不同入口

### 3.1 Legacy group/container

```text
[sysfsdev]/iommu_group
  -> group ID
  -> /dev/vfio/<group>
  -> VFIO_GROUP_GET_DEVICE_FD
  -> shared legacy VFIOContainer
  -> MemoryListener
```

QEMU 先从 sysfs 找到 IOMMU group，再打开 group fd。group 的所有 function
必须满足 kernel 的 viability/ownership 规则。

### 3.2 Device cdev + IOMMUFD

```text
[sysfsdev]/vfio-dev/vfioX/dev
  -> /dev/vfio/devices/vfioX
  -> /dev/iommu backend
  -> VFIO_DEVICE_BIND_IOMMUFD
  -> IOAS alloc/reuse
  -> HWPT alloc/reuse
  -> VFIO_DEVICE_ATTACH_IOMMUFD_PT
  -> MemoryListener
```

QEMU 从一个已经配置的 IOMMUFDBackend 取得 `/dev/iommu` fd，不再通过
legacy group 文件取得 device fd。

### 3.3 Backend 选择

[`vfio_device_attach()`](../../qemu/hw/vfio/device.c:461)根据
`vbasedev->iommufd` 是否存在选择：

- `TYPE_VFIO_IOMMU_LEGACY`
- `TYPE_VFIO_IOMMU_IOMMUFD`

然后调用 backend class 的 `attach_device`。这使 `vfio-pci` 的 PCI 建模代码
不直接写死某一种 kernel VFIO 入口。

## 4. PCI realize 的真实顺序

[`vfio_pci_realize()`](../../qemu/hw/vfio/pci.c:3573)是理解整个方案的主入口。
主要顺序如下：

1. 接受 host BDF、已打开的 fd 或 sysfsdev。
2. `vfio_device_get_name()` 建立稳定设备名。
3. 检测是否 mdev，并限制 balloon/RAM discard 兼容选项。
4. 处理 PCI VF token。
5. `vfio_device_attach()` 建立 Host fd、IOMMU backend 和 memory mapping。
6. `vfio_pci_populate_device()` 读取 region/IRQ/device flags。
7. `vfio_pci_config_setup()` 读取 config，准备 BAR、ROM 和 MSI-X 元数据。
8. 为 physical device 建立 `HostIOMMUDevice`/vIOMMU capability。
9. `vfio_pci_add_capabilities()` 建立 Guest PCI capabilities。
10. 应用 PCI/vendor quirks。
11. 建立 INTx/MSI/MSI-X 路径。
12. 探测 display/ramfb。
13. `vfio_migration_realize()` 查询 migration feature。
14. 注册 AER、reset request 和 CPR hooks。

失败路径按反向顺序拆除 display、IRQ、MSI、BAR、vIOMMU 和 device。PCI
直通的许多故障来自初始化中途成功、后续步骤失败后的清理不完整，因此这个
顺序值得逐段阅读。

## 5. Legacy attach 源码

[`vfio_legacy_attach_device()`](../../qemu/hw/vfio/container-legacy.c:946)：

1. 从 sysfsdev/iommu_group 得到 group ID；
2. `vfio_group_get()` 打开 group 并检查 `VFIO_GROUP_FLAGS_VIABLE`；
3. 建立或复用该 AddressSpace 的 container；
4. `vfio_device_get()` 调 `VFIO_GROUP_GET_DEVICE_FD`；
5. 建立 `HostIOMMUDevice`；
6. mdev 需要额外 migration/CPR blocker。

[`vfio_group_get()`](../../qemu/hw/vfio/container-legacy.c:762)在 group
不可见或不可归属时直接失败。
[`vfio_device_get()`](../../qemu/hw/vfio/container-legacy.c:844)读取
`VFIO_DEVICE_GET_INFO`，记录 region/IRQ 数量和 reset capability，并把设备
加入 container 和全局 VFIO device list。

因此 legacy 路径的共享粒度包括：

- 相同 Guest AddressSpace -> 相同 `VFIOAddressSpace`；
- 兼容 group -> 可共享 VFIO container/IO page table；
- 多 function 的 reset dependency 通过 group fd 传递。

## 6. IOMMUFD attach 源码

### 6.1 打开 cdev

[`iommufd_cdev_attach()`](../../qemu/hw/vfio/iommufd.c:629)如果只给了
sysfsdev，会：

1. 查找 `[sysfsdev]/vfio-dev/vfioX/dev`；
2. 读取 major:minor；
3. 用 `open_cdev()` 打开 `/dev/vfio/devices/vfioX`；
4. 使用 QEMU 配置的 IOMMUFDBackend。

这里没有先打开 group fd。

### 6.2 Bind

[`iommufd_cdev_connect_and_bind()`](../../qemu/hw/vfio/iommufd.c:156)：

1. 连接 QEMU IOMMUFD backend；
2. 把 device fd 注册到 KVM VFIO device；
3. 可选传 VF PCI VF token；
4. `VFIO_DEVICE_BIND_IOMMUFD`；
5. 保存 kernel 返回的 `device_id`。

`device_id` 是当前 iommufd namespace 中的 object ID，不是 Guest BDF。

### 6.3 复用 IOAS/container

QEMU 在同一 AddressSpace 中寻找相同 `IOMMUFDBackend` 的 container：

- 找到则尝试把新 device attach 到已有 domain；
- 不兼容时创建新 IOAS；
- IOAS 成功后建立 QEMU `VFIOIOMMUFDContainer`；
- 创建/复用 HWPT；
- 把 device 或 PASID attach 到 HWPT；
- 注册 MemoryListener；
- 最后读取 `VFIO_DEVICE_GET_INFO`。

代码注释明确要求 `HostIOMMUDevice` 必须在 device/HWPT attach 后建立，因为
nested IOMMU 需要 device ID 和 HWPT ID。

### 6.4 Physical 与 mdev 的分叉

[`iommufd_cdev_attach_container()`](../../qemu/hw/vfio/iommufd.c:536)中：

- physical device 走 auto-domain；尝试复用兼容 HWPT，否则创建；
- mdev 不尝试物理 auto-domain，直接 attach 到 container 的 IOAS。

这不是说 mdev 一定没有硬件隔离，而是它不走“物理 device -> IOMMU nested
domain”的同一发现方式。

### 6.5 HWPT flags

[`iommufd_cdev_autodomains_get()`](../../qemu/hw/vfio/iommufd.c:389)根据
kernel capability 和 vIOMMU 需求决定：

- `IOMMU_HWPT_ALLOC_DIRTY_TRACKING`
- `IOMMU_HWPT_ALLOC_NEST_PARENT`
- `IOMMU_HWPT_ALLOC_PASID`

这些标志直接影响 migration、Guest vIOMMU 和 per-process GPU address space。
QEMU 不因为 UAPI 常量存在就无条件设置，而是先查 HW info。

## 7. Guest AddressSpace 如何变成 IOMMU 映射

### 7.1 AddressSpace 与 container

QEMU 按 `AddressSpace *` 维护 `VFIOAddressSpace`。同一 Guest IOMMU
AddressSpace 下，可复用一个或多个 VFIO container。

```text
Guest physical address / vIOMMU IOVA
  -> MemoryRegionSection
  -> MemoryListener
  -> VFIOContainer
  -> legacy TYPE1 or IOMMUFD IOAS/HWPT
  -> Host IOMMU
```

### 7.2 Region add

[`vfio_listener_region_add()`](../../qemu/hw/vfio/listener.c:482)调用
`vfio_container_region_add()`。核心步骤：

1. 校验 section 是否允许；
2. 计算 IOVA range；
3. 处理 RAM device/MMIO 对齐；
4. 对 Guest vIOMMU 区域注册 IOMMU notifier；
5. 对普通 RAM 计算 Host VA；
6. 调用 backend `dma_map`；
7. 记录启动期错误或运行时 fatal error。

如果 section 是 Guest vIOMMU，QEMU 不能简单把整个 region 当作一段普通 RAM。
它注册 `IOMMU_NOTIFIER_IOTLB_EVENTS`，在 Guest 修改 stage-1 映射时收到
map/unmap，再同步 Host IOMMU。

### 7.3 IOMMUFD map 的两条路径

[`iommufd_cdev_map()`](../../qemu/hw/vfio/iommufd.c:64)选择：

- shared RAM 或 RAM device -> `IOMMU_IOAS_MAP_FILE`；
- 其他映射 -> `IOMMU_IOAS_MAP`。

源码只对显式 `MAP_SHARED` RAMBlock 或 MMIO/RAM device 使用 file mapping。
`MAP_PRIVATE` 可能因 copy-on-write 导致 PFN 与 backing file 永久分叉，因此
不能安全转成 file-backed DMA。

### 7.4 Region delete

[`vfio_listener_region_del()`](../../qemu/hw/vfio/listener.c:663)注销 notifier
并 unmap。若正在做 dirty tracking，会先取得 dirty bitmap，避免 Guest 停止
访问后丢失最后一批 Host 写入。

AI memory hotplug、VM memory resize、virtio-mem/balloon、dma-buf 回收都要
经过这条路径。只验证初始 VM boot 不能证明动态内存生命周期正确。

## 8. BAR、MMIO 与 mmap

### 8.1 Region setup

[`vfio_pci_populate_device()`](../../qemu/hw/vfio/pci.c:3143)检查 PCI flags、
region 数量和 IRQ 数量，然后为 BAR0-5 调
[`vfio_region_setup()`](../../qemu/hw/vfio/region.c:231)。

`VFIORegion`保存：

- VFIO region index；
- size、flags、fd offset；
- `MemoryRegionOps`；
- sparse mmap 区间；
- backing VFIODevice。

### 8.2 Trap 路径

[`vfio_region_read()`](../../qemu/hw/vfio/region.c:91)和
[`vfio_region_write()`](../../qemu/hw/vfio/region.c:38)把 Guest MMIO access
转成 `pread/pwrite` 或 VFIO ioctl fallback，并处理 little endian。

这些函数属于慢路径。例如 BAR 不可 mmap、稀疏区间未映射、QEMU quirk 或
debug access 时，可能发生 QEMU 往返。

### 8.3 Fast mmap 路径

[`vfio_region_mmap()`](../../qemu/hw/vfio/region.c:349)：

1. 读取 region mmap capability；
2. 为 regular/sparse 区间分配固定地址窗口；
3. 从 VFIO device fd `mmap()` Host BAR range；
4. 建立 RAMBlock/MemoryRegion 子区间；
5. 让 Guest BAR 访问尽量直接落到 Host device aperture。

这减少 QEMU trap，但仍不是“Guest 获得物理 BAR pointer”。Guest GPA 仍在
QEMU AddressSpace 中，KVM 把对应 Host mapping 写入 nested page table。

### 8.4 DMA-BUF 与 P2P

固定源码支持向 kernel 请求 `VFIO_DEVICE_FEATURE_DMA_BUF`，把 region mapping
转成 RAMBlock 关联的 dma-buf fd；kernel 不支持或设备不支持时退回普通 mmap，
并明确提示 BAR P2P 可能不可用。

这对 GPU/NIC P2P 很关键：

```text
dma-buf-capable VFIO region
  -> exportable/importable DMA object
  -> possible peer mapping

plain mmap fallback
  -> CPU-visible mapping may work
  -> device-to-device P2P may fail
```

不能只因为 BAR 可读写就推断 NVLink/XGMI/PCIe P2P 可用。

## 9. PCI config space

[`vfio_pci_config_setup()`](../../qemu/hw/vfio/pci.c:3408)读取 Host config
space，然后建立 QEMU 的 emulated bits：

- ROM enable/address 由 QEMU 控制；
- BAR address/type 由 QEMU PCI allocator 控制；
- vendor/device/subsystem/class 可配置覆盖；
- multifunction bit 由 QEMU topology 决定；
- Host resource address 被清空后由 Guest PCI enumeration 重新分配。

因此 VFIO 是“物理 config 的受控投影”，不是把整个 Host config space 原样
暴露。GPU 驱动看到的 PCI header、capability chain 和 BAR layout 由 Linux
VFIO 与 QEMU 共同构造。

### 9.1 对 AI 驱动的重要性

CUDA/ROCm 会枚举：

- vendor/device/subsystem；
- BAR0/2/4；
- MSI-X；
- AER/PM/SR-IOV；
- ROM 和 serial number；
- GPU/NVLink capability。

任一字段与本机 vendor driver 期望不一致，都可能在 `probe` 早期失败。

## 10. 中断路径

### 10.1 INTx

QEMU 支持两种模式：

- 用户态：VFIO eventfd 唤醒 QEMU，调用 `pci_irq_assert/deassert`；
- KVM 加速：IRQFD + resamplefd，把 Host eventfd 接到 Guest GSI。

[`vfio_intx_enable_kvm()`](../../qemu/hw/vfio/pci.c:154)建立 resample/unmask
路径。该路径需要 KVM IRQFD/resamplefd 支持，否则退回 QEMU handler。

### 10.2 MSI/MSI-X

[`vfio_pci_interrupt_setup()`](../../qemu/hw/vfio/pci.c:3535)将 Guest MSI 和
MSI-X capability 的相关 config bits 标记为 QEMU emulated。

MSI/MSI-X enable 时，QEMU：

1. 为 Guest vector 建立 eventfd；
2. 配置 KVM MSI route；
3. `VFIO_DEVICE_SET_IRQS` 把 Host interrupt 绑到 eventfd；
4. eventfd 被触发时直接注入 Guest 或回到 QEMU handler。

MSI-X masked vector 触发时，QEMU 设置 pending bit 并启用 PBA MMIO，Guest
unmask 后补送中断。MSI-X table/PBA 的 BAR 归属、mask 状态和 Host kernel
view 必须一致。

### 10.3 为什么 doorbell 与 interrupt 不同

GPU admin/RAS 使用 MSI-X；
compute completion 常用 memory write + fence；
Host-to-device submission 使用 MMIO doorbell。

doorbell 通常走 BAR fast mmap 或 `IOEVENTFD`，completion interrupt走 MSI-X。
把所有通知都叫作“interrupt”会掩盖真实的延迟路径。

## 11. Reset 路径

[`vfio_pci_reset()`](../../qemu/hw/vfio/pci.c:3765)是组合式 reset：

1. 执行 QEMU `vfio_pci_pre_reset()`；
2. reset display；
3. 尝试 device/vendor reset quirk；
4. 尝试 `VFIO_DEVICE_RESET`（可能为 FLR）；
5. 尝试单设备 hot reset；
6. 最后尝试 PM reset；
7. 执行 `vfio_pci_post_reset()` 恢复 MSI-X/routing/device state。

`vfio_device_reset_handler()` 在系统 reset 时扫描全局设备，并区分：

- 一个被使用的 VFIO device；
- 同一 hot-reset domain 中多个被使用的 VFIO device。

legacy backend 必须把相关 group fd 传给
`VFIO_DEVICE_PCI_HOT_RESET`；IOMMUFD backend 使用 device ID 判断依赖
ownership。未拥有依赖设备时，bus reset 被拒绝。

### 11.1 AI 故障域

PF/firmware hang 时通常需要更大的 reset scope。一个 GPU VF 或 GPU function
执行 FLR，可能影响：

- 同一 slot/function group；
- PF 管理的 firmware scheduler；
- 其他 VF 的 queue；
- P2P link 状态；
- collective 中等待该 GPU 的 rank。

QEMU 成功完成 ioctl 不等于业务进程已经恢复。Guest KMD 还需要重建 GPU VM、
queue、MSI-X 和 fence。

## 12. Migration 路径

### 12.1 能力发现

[`vfio_migration_query_flags()`](../../qemu/hw/vfio/migration.c:1117)通过
`VFIO_DEVICE_FEATURE_MIGRATION` 读取：

- STOP_COPY；
- P2P；
- PRE_COPY；
- v2 pre-copy info。

`vfio_dma_logging_supported()` 另外探测 device DMA logging。

### 12.2 Save

[`vfio_save_setup()`](../../qemu/hw/vfio/migration.c:589)：

1. 建立 multifd channel（如使用）；
2. 查询 STOP_COPY data size；
3. 分配 bounded buffer；
4. 若支持，从 RUNNING 进入 PRE_COPY；
5. 查询后续 pending size。

[`vfio_save_iterate()`](../../qemu/hw/vfio/migration.c:751)搬运 device opaque
stream，并维护初始/脏数据计数。
[`vfio_save_complete_precopy()`](../../qemu/hw/vfio/migration.c:779)进入
STOP_COPY 后完整取流。

### 12.3 Load

incoming QEMU 在 device open/attach 后进入 RESUMING，读取 stream，最后转到
STOP/RUNNING。源码还保存 PCI config、interrupt routing 和 vendor migration
format。

### 12.4 内存 dirty tracking

VFIO migration 至少需要两条 dirty 来源：

- Guest CPU 写 Guest RAM；
- Device DMA 写 Guest RAM。

container 的 dirty tracking 根据 IOMMU HWPT 与 device DMA logging 能力选择
backend。QEMU 在 unmap 时同步最后一次 bitmap，防止漏迁。

### 12.5 为什么会失败

常见兼容性门：

- device 只实现 STOP_COPY，没有 PRE_COPY；
- P2P 不能进入 quiescent state；
- dirty tracking 不覆盖某些 BAR/P2P 路径；
- QEMU 禁止 postcopy 或 background snapshot；
- source/destination GPU/firmware 不兼容；
- Guest 正在运行 long kernel 或 collective；
- SR-IOV VF 的 PF migration 状态不匹配。

“VFIO migration 已配置”不等于某个 GPU profile 可以 live migration。

## 13. QEMU VFIO 与 virtio-gpu 的源码边界

| 维度 | QEMU VFIO | QEMU virtio-gpu |
| --- | --- | --- |
| Guest PCI device | Host function 的受控投影 | 模拟 `1af4:1050` |
| Guest driver | 厂商原生 KMD | virtio-gpu KMD + capset UMD |
| QEMU 是否理解 GPU command | 通常不理解 | 不解析高层语义，但处理 virtio capset |
| BAR | Host BAR 映射/trap | virtio BAR、resource、shared region |
| 提交路径 | Guest 直接写 doorbell/ring | virtqueue、blob/ring/renderer |
| Host 执行 | 物理 GPU 与 Guest vendor driver | Host renderer/vendor driver |
| 中断 | Host MSI-X -> eventfd/KVM route | virtio interrupt、fence response |
| 隔离边界 | IOMMU + physical/virtual function | renderer parser + Host API/GPU |
| 迁移 | VFIO migration UAPI + vendor state | QEMU device/renderer-specific |

virtio-gpu 的 QEMU 设备拥有更多设备语义：它需要解析 virtio header、管理
resource/context/fence、调用 VirGL/Venus/GFXStream/Rutabaga。VFIO 的 QEMU
路径尽量少解析 GPU command，但必须完整实现 DMA/IRQ/PCI/reset 和迁移边界。

## 14. AI workload 的 QEMU 侧关注点

### 14.1 启动时间

大 BAR、dma-buf、MSI-X table、多项 vIOMMU capability 和 vendor quirk 都在
realize 阶段完成。AI 单卡启动慢不一定是 Guest driver 慢。

### 14.2 大内存 pin/map

QEMU MemoryListener 把大块 Guest RAM 映射给 IOMMU。HBM 模型不一定在 Guest
RAM 中，但权重上传、KV cache overflow 和多 GPU staging buffer 会经过这些页。
动态 resize 和 balloon 必须触发正确 unmap/dirty capture。

### 14.3 P2P

BAR 普通 mmap 与 device dma-buf P2P 是不同能力。NCCL/RCCL topology 与
peer access 可能因 QEMU mapping 方式、IOMMU nested domain 或 ACS 而改变。

### 14.4 MSI-X 数量

compute/RDMA GPU 可能有大量 queue/vector。MSI-X table size、BAR 映射和
KVM route 数量限制会在多进程、多 stream workload 下放大。

### 14.5 migration

QEMU 的 migration thread 只负责搬运 opaque stream。VRAM 内容、GPU page
table、queue state 和 collective 必须由 vendor driver 和 workload 共同处理。

## 15. 失败矩阵

| 症状 | QEMU 侧优先检查 |
| --- | --- |
| `vfio-pci` realize 失败 | sysfsdev、group viability、iommufd fd、bind/attach |
| cdev 不存在 | sysfs `vfio-dev/vfioX`、`CONFIG_VFIO_DEVICE_CDEV` |
| IOMMUFD bind 失败 | 第二 opener、group ownership、VF token |
| DMA map 失败 | AddressSpace、IOVA range、alignment、MAP_FILE 条件 |
| Guest BAR read 返回全 1 | region setup、Host Memory Space、runtime PM |
| BAR 很慢 | 是否落到 `vfio_region_read/write` trap |
| GPU 中断不投递 | IRQFD/MSI route、mask/PBA、eventfd |
| GPU 能探测但 workload 失败 | config/BAR 能力、Guest vendor driver ABI |
| P2P 失败 | dma-buf/region mmap fallback、IOMMU/ACS |
| reset 失败 | FLR、hot reset dependency、未拥有 function |
| reset 成功但 Guest KMD 坏 | post-reset queue/VM/fence 重建 |
| migration setup 失败 | VM state、feature flags、dirty tracking |

## 16. 推荐源码阅读顺序

1. `hw/vfio/pci.c`：`vfio_pci_realize()` 和 teardown。
2. `hw/vfio/device.c`：backend 选择、IRQ helper、全局 device list。
3. `hw/vfio/container-legacy.c`：group/container 路径。
4. `hw/vfio/iommufd.c`：cdev、bind、IOAS/HWPT attach。
5. `hw/vfio/listener.c`：Guest AddressSpace -> IOVA/DMA map。
6. `hw/vfio/region.c`：BAR trap、mmap、sparse mmap、dma-buf。
7. `hw/vfio/pci.c` 的 config、MSI-X、reset 部分。
8. `hw/vfio/migration.c`：pre-copy、STOP_COPY、dirty tracking。
9. `hw/vfio/display.c`：GPU display/ramfb。
10. `hw/vfio/pci-quirks.c`、`igd.c`：vendor-specific behavior。

## 17. 检查题

1. 为什么 QEMU VFIO 需要同时理解 Host IOMMU backend 和 Guest PCI 建模？
2. legacy path 在哪里取得 device fd，IOMMUFD path 在哪里取得？
3. `VFIORegion` 的 trap 路径和 mmap fast path 分别何时使用？
4. 为什么 Guest BAR 可写不能证明 P2P 可用？
5. QEMU 为什么模拟 PCI config/BAR/MSI-X 的部分字段？
6. INTx KVM resamplefd 解决什么问题？
7. MSI-X vector masked 时为什么需要 PBA？
8. Physical device 和 mdev 在 IOMMUFD auto-domain 上如何分叉？
9. IOAS/HWPT attach 为什么要早于 `HostIOMMUDevice` realize？
10. QEMU 如何在 vIOMMU 动态映射时收到 map/unmap 事件？
11. VFIO reset 为什么可能要求多个 function/group 一起参与？
12. VFIO migration 的 opaque stream、Guest RAM 和 GPU VRAM 分别由谁搬运？
13. 为什么 dirty tracking 必须覆盖 Host device DMA？
14. QEMU VFIO 和 QEMU virtio-gpu 在“是否解析 GPU command”上差在哪里？
15. AI 推理 decode 与训练 collective 分别会暴露哪些 QEMU 侧边界？

## 18. 与其他专题

- [Linux VFIO/IOMMUFD/SR-IOV/mdev 源码](vfio-iommufd-sriov-source-trace.md)
- [PCIe、IOMMU、ATS/PASID 与 GPU P2P](pcie-iommu-ats-p2p.md)
- [GPU VM 生命周期、热插拔、Checkpoint 与迁移](vm-lifecycle-migration.md)
- [GPU 执行模型、队列、调度与故障隔离](gpu-execution-model.md)
- [Virtio-GPU 四路径源码横向对照](../virtio-gpu/source-trace-cross-path.md)
- [GPU 虚拟化平台比较](../virtio-gpu/platform-comparison.md)
