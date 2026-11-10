# ARM IOMMU 与 Xen Dom0 设备直通：方案与源码流程

## 1. 文档定位

本文回答三个问题：

1. ARM 架构下的 IOMMU/SMMU 如何把设备 DMA 地址翻译到物理内存？
2. Xen 如何让一个物理设备由 Dom0 直接驱动？
3. 这条路径在 Xen 源码中从启动探测到 DMA 映射、失效和重新分配分别经过
   哪些函数？

本文只做源码和文档梳理，不启动 Xen、不修改设备树、不绑定设备，也不运行
直通实验。

更详细的硬件原理解释，以及 Linux 内核与 Xen SMMU 驱动的逐项源码对照，见
[ARM SMMU 原理与 Linux/Xen 源码对照](smmu-principles-linux-xen.md)。

源码基线：

```text
Xen 4.23-dev-454-gdf8d101a38
xen.git commit df8d101a3854725fd3db3c52f074808e6b3cc539
Linux commit 6b4b543fa18757297ac756a6fe6e64f506b7844a
```

`xen/` 工作树当前有少量未提交的 x86 文件修改；本文引用的 ARM 和
`drivers/passthrough/` 文件没有出现在这些修改中。

## 2. 先给结论

### 2.1 Xen Dom0 的“直通”首先是硬件域设备所有权

ARM Xen 没有要求用户先用 Linux VFIO 把设备从 Dom0 解绑，再通过 QEMU
`vfio-pci` 交给 Domain。Dom0 是 hardware domain，绝大多数未被 Xen
保留的设备默认就是 Dom0 的设备：

```text
Host Device Tree
  -> Xen 检查设备是否标记为 xen,passthrough
  -> 没有标记：设备归 Dom0
  -> 受 Xen 管理的 SMMU 保护：Xen 自动把设备 attach 到 Dom0 的 SMMU
     translation domain
  -> 为 Dom0 建立 MMIO、IRQ 和 Device Tree 视图
  -> Dom0 Linux 用原生设备驱动直接操作硬件
```

因此，从 Xen hypervisor 视角看，Dom0 的设备直通主要是：

- 设备所有权归 Dom0；
- Dom0 的 P2M 中可见设备 MMIO；
- 设备 IRQ 路由到 Dom0；
- 受 SMMU 保护的设备使用 Dom0 的 P2M 作为 stage-2 翻译表；
- Dom0 获得一个隐藏了由 Xen 管理的 SMMU 信息的设备树。

### 2.2 `xen,passthrough` 不是“给 Dom0 开启直通”

在 Xen ARM 源码和文档中，`xen,passthrough` 的含义是：

> 这个设备要从 Dom0 隐藏，预留给后续的非 Dom0 guest。

相关文档明确写着，带该属性的设备不会暴露给 Dom0
（[passthrough.txt](../../xen/docs/misc/arm/device-tree/passthrough.txt:4)）。
代码在 `write_properties()` 中把该设备的 `status` 改成 `"disabled"`，并且
`handle_device()` 不会为它建立 Dom0 的 MMIO/IRQ 映射
（[domain_build.c](../../xen/xen/arch/arm/domain_build.c:549)，
[status override](../../xen/xen/arch/arm/domain_build.c:588)，
[device.c](../../xen/xen/arch/arm/device.c:248)）。

如果目标是“让 Dom0 直接驱动物理设备”，通常不应在 host Device Tree 上给
这个设备加 `xen,passthrough`。

### 2.3 Xen 使用的是 SMMU stage-2，而不是 Linux VFIO 模型

当前 Xen ARM SMMU 路径只使用 stage-2 translation。设备提交的 DMA 地址在
逻辑上被解释成目标 domain 的 IPA（intermediate physical address），SMMU
使用这个 domain 的 P2M 将其翻译为 HPA（host physical address）。

SMMUv2 驱动把 Xen 的 P2M 根页表写入 context bank 的 TTBR0
（[smmu.c](../../xen/xen/drivers/passthrough/arm/smmu.c:1250)）。
SMMUv3 驱动把 P2M 根地址写入 Stream Table Entry 指向的 stage-2
`VTTBR`，并生成对应 `VTCR`
（[smmu-v3.c](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:1205)）。

所以 Xen 不再维护一张独立的、与 CPU P2M 分离的普通 IOMMU 页表。CPU 和
IOMMU 共享 P2M 的结构是理解整条 Dom0 直通路径的核心。

### 2.4 SMMU 提供的是 domain 级 DMA 边界

一个 Xen domain 在某个 SMMU 上通常对应一个 IOMMU translation domain。该
domain 内多个设备共享同一 stage-2 地址空间。对于 Dom0：

- 设备不能任意访问 Xen 或 DomU 的物理页；
- 设备可以访问 Dom0 P2M 中允许它翻译到的地址；
- SMMU 不会把一个 Dom0 设备自动限制为只能访问它自己的 BAR；
- SMMU 不负责设备 reset、固件隔离、GPU 内部 context 隔离或 cache
  coherence。

因此，“设备已 attach 到 SMMU”证明 DMA 经过受控的 stage-2 翻译，不等于
设备的所有硬件子资源和故障域都已经隔离。

## 3. ARM IOMMU/SMMU 模型

### 3.1 IOMMU 在系统中的位置

没有 IOMMU 时，设备发起的总线地址通常直接落到物理地址空间。有 SMMU 时：

```text
Device driver
  -> device DMA address
  -> bus / interconnect
  -> StreamID / Requester ID
  -> ARM SMMU
  -> IPA -> HPA translation
  -> memory controller
```

这里的“device DMA address”是设备实际提交到总线上的地址，不应简单等同于
CPU 虚拟地址。在 Xen passthrough domain 中，它通常对应 domain 的 IPA。

### 3.2 为什么设备身份不能只看地址

SMMU 必须先区分“是谁发起了这次 DMA”，然后才能选择正确的翻译上下文。
ARM SMMU 的主要身份信息是 StreamID：

- 平台设备：通常由 Device Tree 的 `iommus = <&smmu STREAM_ID>` 描述；
- PCI 设备：通常由 PCI host bridge 的 `iommu-map` 把 Requester ID/BDF
  映射到 StreamID。

两个设备即使使用相同的 DMA 地址，只要 StreamID 不同，也可以进入不同的
translation context。

### 3.3 SMMUv2 的结构

SMMUv2 的关键硬件对象包括：

| 对象 | 作用 |
| --- | --- |
| SMR | Stream Match Register，匹配 StreamID |
| S2CR | Stream-to-Context Register，把匹配结果指向 context bank 或 bypass/fault |
| Context Bank | 保存一个翻译上下文的配置和页表根 |
| TTBR0/TTBCR | 指定页表根和翻译参数 |
| VMID | 区分 stage-2 地址空间 |

简化路径：

```text
StreamID
  -> SMR stream matching
  -> S2CR
  -> Context Bank
  -> stage-2 page table
  -> HPA
```

Xen 的 SMMUv2 代码在 `arm_smmu_master_alloc_smes()` 中分配 SMR/S2CR
条目（[smmu.c](../../xen/xen/drivers/passthrough/arm/smmu.c:1630)），在
`arm_smmu_domain_add_master()` 中把 S2CR 指向该 domain 的 context bank
（[smmu.c](../../xen/xen/drivers/passthrough/arm/smmu.c:1679)）。

`arm_smmu_init_context_bank()` 明确把 `CBAR` 设为 stage-2 translation，
随后使用：

```text
p2maddr = page_to_maddr(domain->arch.p2m.root)
```

作为 context bank 的页表根
（[smmu.c](../../xen/xen/drivers/passthrough/arm/smmu.c:1157)）。

### 3.4 SMMUv3 的结构

SMMUv3 用内存中的表替代了 SMMUv2 大量 MMIO stream mapping 寄存器：

| 对象 | 作用 |
| --- | --- |
| Stream Table | 以 StreamID 查找 Stream Table Entry |
| STE | 配置 bypass、abort、stage-1 或 stage-2 翻译 |
| S2VMID | stage-2 地址空间标识 |
| VTCR/VTTBR | stage-2 翻译参数和根表 |
| Command Queue | 提交 CFGI/TLBI 等维护命令 |
| Event Queue | 报告 translation fault 等事件 |
| PRI Queue | 处理 PCIe PRI 相关请求 |

简化路径：

```text
StreamID
  -> Stream Table
  -> STE (S2_TRANS)
  -> S2VMID + VTCR + VTTBR
  -> domain P2M
  -> HPA
```

`arm_smmu_domain_finalise_s2()` 把 `smmu_domain->d->arch.p2m.root` 转成
物理地址作为 `VTTBR`，并以 `p2m_ipa_bits` 计算 `VTCR.T0SZ`
（[smmu-v3.c](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:1163)）。
`arm_smmu_write_strtab_ent()` 最终写出 `STRTAB_STE_0_CFG_S2_TRANS`
配置
（[smmu-v3.c](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:666)）。

SMMUv3 驱动在页表更新后通过 command queue 发出 `CFGI`、`TLBI` 和
`SYNC` 命令，并同步对应 translation/TLB cache。

### 3.5 Xen 为什么只需要 stage-2

Xen 自己管理 domain 的 P2M，因此可以直接把 P2M 作为 SMMU 的 stage-2：

```text
Dom0 driver 配置设备 DMA 地址 = IPA
  -> SMMU stage-2 用 Dom0 P2M 翻译 IPA
  -> HPA
```

这条路径与 CPU 在 EL2 下执行 stage-2 翻译使用同一个地址空间模型。对
passthrough guest 来说，guest 原生驱动不需要知道 Host IOVA 的存在。

SMMUv2 驱动把 `force_stage` 固定为 2
（[smmu.c](../../xen/xen/drivers/passthrough/arm/smmu.c:2682)）；
SMMUv3 在 domain finalise 时直接把 stage 设为 `ARM_SMMU_DOMAIN_S2`
（[smmu-v3.c](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:1231)）。

这不是 ARM SMMU 硬件只能做 stage-2，而是当前 Xen 的 passthrough 实现选择
了单层 stage-2。Guest-managed stage-1 加 Host-managed stage-2 的 vSMMU
嵌套模型不属于本文这条 Dom0 路径。

### 3.6 地址链

在 Dom0 中应区分以下视图：

```text
Dom0 CPU VA
  -> Dom0 stage-1 page table
  -> IPA
  -> Xen P2M (CPU stage-2)
  -> HPA

Device DMA address
  -> SMMU stage-2
  -> same Xen P2M
  -> HPA
```

典型 Dom0 是 direct-mapped domain，很多 IPA 和 HPA 的数值相同，设备 MMIO
也常以 1:1 地址映射给 Dom0。但概念上仍应先按 IPA 理解，不能在所有平台、
所有内存布局和所有 SMMU 配置上把两者无条件等同。

当前 ARM Xen passthrough 核心代码主要解析 `iommus` 和 `iommu-map`，没有
在这条路径中额外实现通用的 `dma-ranges` 地址窗口转换。平台如果存在复杂
总线地址偏移，必须根据实际 Device Tree、硬件连接和驱动的 DMA 地址语义单独
确认，不能只凭“有 `iommus`”推断地址关系。

## 4. Device Tree 如何描述直通设备

### 4.1 一个概念性 host Device Tree

```dts
smmu0: iommu@2b400000 {
    compatible = "arm,smmu-v3";
    reg = <0x0 0x2b400000 0x0 0x20000>;
    #iommu-cells = <1>;
    dma-coherent;
};

ethernet@ff0e0000 {
    compatible = "vendor,ethernet";
    reg = <0x0 0xff0e0000 0x0 0x1000>;
    interrupts = <GIC_SPI 63 IRQ_TYPE_LEVEL_HIGH>;
    iommus = <&smmu0 0x1234>;
};
```

关键点：

- `#iommu-cells = <1>` 表示该 SMMU 的每个 master specifier 有一个 cell；
- 对 SMMUv2/v3，这个 cell 通常是 StreamID；
- `iommus` 表明设备是 SMMU master；
- 设备节点本身仍描述 MMIO 和 IRQ，`iommus` 只建立设备身份到 SMMU
  context 的关系；
- SMMU 节点由 Xen 初始化后需要从 Dom0 视图中隐藏。

具体 binding 见：

- [通用 IOMMU binding](../../linux/Documentation/devicetree/bindings/iommu/iommu.txt:1)
- [ARM SMMU binding](../../linux/Documentation/devicetree/bindings/iommu/arm,smmu.yaml:1)
- [ARM SMMUv3 binding](../../linux/Documentation/devicetree/bindings/iommu/arm,smmu-v3.yaml:1)
- [ARM SMMUv2 官方规范 IHI 0062 2.0 D.c](https://support.arm.com/documentation/ihi0062/dc/)
- [ARM SMMUv3 官方规范 IHI 0070 H.a](https://support.arm.com/documentation/ihi0070/ha/)

规范的版本、PDF 链接、重点章节和代码对象映射见
[ARM SMMU 原理与 Linux/Xen 源码对照](smmu-principles-linux-xen.md:48)。

### 4.2 PCI 设备

PCI endpoint 通常没有自己的静态 `iommus` 属性。PCI host bridge 用
`iommu-map` 把 Requester ID/BDF 区间映射成 SID：

```dts
pcie@40000000 {
    compatible = "vendor,pcie-host";
    device_type = "pci";
    reg = <0x0 0x40000000 0x0 0x100000>;
    #address-cells = <3>;
    #size-cells = <2>;
    ranges = <...>;

    iommu-map = <
        0x0000 &smmu0 0x10000 0x10000
    >;
};
```

含义是：host bridge 下的 BDF 通过 `iommu-map` 得到 SID，再由 SMMU 选择
STE/context。Xen 的 `iommu_add_dt_pci_sideband_ids()` 读取这个映射，并把
SID 填入 PCI device 的 `iommu_fwspec`
（[device_tree.c](../../xen/xen/drivers/passthrough/device_tree.c:165)）。

## 5. Xen 启动与 IOMMU 初始化

### 5.1 启动顺序

[`start_xen()`](../../xen/xen/arch/arm/setup.c:281) 的顺序与本文有关的
部分是：

```text
解析并展开 host Device Tree
  -> 初始化内存、中断、timer、GIC 和体系结构
  -> iommu_setup()
  -> setup_virt_paging()
  -> do_initcalls()
  -> create_dom0()
```

源码注释说明了为什么 IOMMU 在 P2M 前初始化：

- Xen 必须先从 SMMU 驱动收集该 SMMU 支持的最大 IPA 位宽；
- 所有 SMMU 的约束取最小值后，才能配置 CPU stage-2 的 VTCR 和 P2M 根；
- 这样 CPU 和 SMMU 看到的是兼容的 stage-2 地址宽度。

### 5.2 `iommu_setup()`

[`iommu_setup()`](../../xen/xen/drivers/passthrough/iommu.c:550) 负责：

1. 根据 Xen command line 解析 `iommu=` 和 `dom0-iommu=`；
2. 调用体系结构相关的 `iommu_hardware_setup()`；
3. 成功时取全局 `iommu_ops` 并设置 `iommu_enabled`；
4. 初始化 quarantine domain；
5. 打印 `I/O virtualisation enabled` 和 Dom0 模式。

ARM 的 [`iommu_hardware_setup()`](../../xen/xen/drivers/passthrough/arm/iommu.c:51)
遍历 host Device Tree，对每个节点调用：

```text
device_init(np, DEVICE_IOMMU, NULL)
```

只有匹配已注册 IOMMU 驱动的节点才会初始化。匹配到 SMMUv2/v3 后，该驱动：

1. 调用 `iommu_set_ops()` 注册全局操作集；
2. 探测 MMIO、IRQ、stream table、context bank 和硬件能力；
3. 读取 SMMU 的 stage-2 input/output 地址宽度；
4. 调用 `p2m_restrict_ipa_bits()` 收紧 Xen P2M IPA 位宽；
5. 把 SMMU 节点标记为 `DOMID_XEN`，避免交给 Dom0。

入口分别是：

- [`arm_smmu_dt_init()` / SMMUv2](../../xen/xen/drivers/passthrough/arm/smmu.c:2981)
- [`arm_smmu_dt_init()` / SMMUv3](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:2872)

SMMUv3 probe 的硬件初始化和队列/Stream Table 设置在
[`arm_smmu_device_probe()`](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:2477)；
SMMUv2 的对应路径从
[`arm_smmu_device_dt_probe()`](../../xen/xen/drivers/passthrough/arm/smmu.c:2442)
开始。

SMMUv3 驱动中还存在 ACPI/IORT 节点读取代码，但本版本的通用
`iommu_hardware_setup()` 仍是 Device Tree 节点驱动，且 DT 设备注册宏是主要
入口。因此本文把 DT 作为完整主路径，不把“代码中存在 ACPI probe 辅助函数”
等同于“当前树已经具备完整的 ACPI IOMMU 枚举和 Dom0 直通路径”。

### 5.3 `setup_virt_paging()`

[`setup_virt_paging()`](../../xen/xen/arch/arm/mmu/p2m.c:1694) 在所有 SMMU
上报能力之后运行：

1. 比较 CPU 支持的物理地址宽度和 SMMU 能接受的 IPA 位宽；
2. 选择 `p2m_ipa_bits`、VTCR 根层级和 root order；
3. 初始化 VMID allocator；
4. 打印 P2M 位宽和层级。

随后创建 domain 时，
[`p2m_init()`](../../xen/xen/arch/arm/mmu/p2m.c:1506) 分配 P2M 根页表，
由 [`p2m_alloc_table()`](../../xen/xen/arch/arm/mmu/p2m.c:1427) 生成
`p2m->root` 和 `p2m->vttbr`。

## 6. 创建 Dom0 时 IOMMU 做什么

### 6.1 Domain 创建顺序

[`create_dom0()`](../../xen/xen/arch/arm/domain_build.c:1971) 设置：

```text
XEN_DOMCTL_CDF_hvm
XEN_DOMCTL_CDF_hap
XEN_DOMCTL_CDF_trap_unmapped_accesses
```

如果全局 IOMMU 已启用，还设置：

```text
XEN_DOMCTL_CDF_iommu
```

然后调用 `domain_create()`。ARM 的
[`arch_domain_create()`](../../xen/xen/arch/arm/domain.c:665) 先调用
[`iommu_domain_init()`](../../xen/xen/drivers/passthrough/iommu.c:196)，
再调用 `p2m_init()`。

这个顺序很重要：

1. `iommu_domain_init()` 调用平台 SMMU 的 `init`；
2. SMMU `init` 为 Xen domain 建立 `arm_smmu_xen_domain`，初始化
   `contexts` 链表；
3. `p2m_init()` 随后分配 P2M 根；
4. 真正 attach 设备时，SMMU driver 再读取已经存在的 P2M 根，完成 context
   bank 或 STE 的配置。

### 6.2 Dom0 的通用 IOMMU 初始化

[`construct_hwdom()`](../../xen/xen/arch/arm/domain_build.c:1921) 首先调用
`iommu_hwdom_init(d)`。ARM 的
[`arch_iommu_hwdom_init()`](../../xen/xen/drivers/passthrough/arm/iommu.c:136)
禁用 ARM 不支持的 `map-inclusive` 和 `map-reserved` 选项。

之后才：

1. 为 Dom0 分配内存；
2. 处理共享内存和额外平台映射；
3. 准备 Dom0 Device Tree；
4. 逐个设备建立 MMIO、IRQ 和 SMMU 关系；
5. 装载 kernel/initrd/DTB 并启动 Dom0。

因此，Dom0 的设备认领发生在
[`prepare_dtb_hwdom()`](../../xen/xen/arch/arm/domain_build.c:1686) 的递归
遍历中，而不是在 `iommu_hwdom_init()` 中。

## 7. Dom0 设备认领的源码主流程

### 7.1 遍历 host Device Tree

`prepare_dtb_hwdom()` 调用
[`handle_node()`](../../xen/xen/arch/arm/domain_build.c:1470) 从 host DT
根开始逐节点生成 Dom0 DT。

与设备直通有关的过滤包括：

- 跳过 Xen 自身的 `/chosen`、PSCI、timer、memory 等特殊节点；
- 跳过 `used_by == DOMID_XEN` 的设备；
- 跳过 `DEVICE_IOMMU`，即 Xen driver 已认领的 SMMU；
- 跳过使用 PPI 的设备；
- 对普通设备调用 `handle_device(d, node, p2mt, ...)`；
- 最后复制或改写设备属性到 Dom0 DT。

### 7.2 `handle_device()`：判断设备是否归 Dom0

[`handle_device()`](../../xen/xen/arch/arm/device.c:248) 使用：

```c
bool own_device = !dt_device_for_passthrough(dev);
```

这里的命名容易误导：

- 没有 `xen,passthrough`：`own_device == true`，设备归当前创建中的
  hardware domain，即 Dom0；
- 有 `xen,passthrough`：`own_device == false`，设备从 Dom0 隐藏，供后续
  guest assignment 使用。

对 `own_device == true` 的设备，处理顺序是：

```text
iommu_add_dt_device(dev)
  -> 解析 iommus / iommu-map
  -> 建立 device IOMMU fwspec
  -> 驱动 add_device
  -> 若设备受 SMMU 保护，标记 dt_device_is_protected

if protected:
    iommu_assign_dt_device(d, dev)
      -> 选择/创建当前 Xen domain 的 SMMU translation domain
      -> attach device

map_device_irqs_to_domain()
map_range_to_domain() for each MMIO range
```

### 7.3 `iommu_add_dt_device()`：把设备身份加入 IOMMU

[`iommu_add_dt_device()`](../../xen/xen/drivers/passthrough/device_tree.c:258)
做的是“注册设备 master 身份”，而不是“把设备交给某个 domain”：

1. 若 IOMMU 未启用，返回正值；
2. 若设备已经有 `iommu_fwspec`，直接成功；
3. 解析设备节点所有 `iommus` 项；
4. [`iommu_dt_xlate()`](../../xen/xen/drivers/passthrough/device_tree.c:140)
   建立 `iommu_fwspec`，绑定 SMMU 设备；
5. 调用驱动 `dt_xlate`，把 specifier 解释为 SID/stream match 信息；
6. 调用驱动 `add_device`。

返回值约定很关键：

| 返回值 | 含义 |
| --- | --- |
| `0` | 设备需要且已经受 IOMMU 保护 |
| `< 0` | 设备需要保护，但注册失败 |
| `> 0` | IOMMU 不存在、未启用，或设备不连接 IOMMU |

SMMUv3 的 `dt_xlate` 把 `iommus` 的一个 cell 直接作为 SID
（[smmu-v3.c](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:1606)）。
SMMUv2 还会处理 StreamID 和可选 mask
（[smmu.c](../../xen/xen/drivers/passthrough/arm/smmu.c:972)）。

### 7.4 `add_device`：建立 master 对象并标记保护

SMMUv3 [`arm_smmu_add_device()`](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:1496)
主要做：

1. PCI 设备先补充 sideband ID；
2. 从 `iommu_fwspec` 找到 SMMU；
3. 分配 `arm_smmu_master`；
4. 检查 SID 在 Stream Table 范围内；
5. 必要时初始化二级 Stream Table；
6. 为非 PCI 平台设备调用 `dt_device_set_protected()`；
7. 记录 SID 列表和 master 对象。

`dt_device_set_protected()` 是后续区分“普通 Dom0 设备”和“需要 attach
SMMU 的设备”的开关。

### 7.5 `iommu_assign_dt_device()`：真正绑定 domain

[`iommu_assign_dt_device()`](../../xen/xen/drivers/passthrough/device_tree.c:29)
的关键检查是：

- domain 必须启用 IOMMU；
- `dt_device_is_protected()` 必须为真；
- 设备不能已经在某个 domain 的 `dt_devices` 链表中；
- 平台 `assign_device` 成功后才把设备挂到当前 domain 的链表；
- 最后设置 `dt_device_set_used_by(dev, d->domain_id)`。

伪代码：

```text
lock(dtdevs)
if dev already in a domain:
    return -EBUSY

rc = hd->platform_ops->assign_device(d, 0, dev, 0)
if rc:
    return rc

list_add(dev, hd->dt_devices)
dev.used_by = d->domain_id
unlock
```

这一步仍不创建 MMIO 映射。MMIO 在 `handle_device()` 返回前的后面几步中
通过 `map_range_to_domain()` 建立。

### 7.6 MMIO 和 IRQ

对 Dom0 普通设备，[`map_range_to_domain()`](../../xen/xen/arch/arm/device.c:40)
做两件事：

1. `iomem_permit_access()` 授权 Dom0 管理该物理 MMIO；
2. 如果不需要跳过映射，`map_regions_p2mt()` 把物理地址按 1:1 方式映射到
   Dom0 P2M。

`prepare_dtb_hwdom()` 传入的默认 MMIO 类型是
`p2m_mmio_direct_c`
（[domain_build.c](../../xen/xen/arch/arm/domain_build.c:1688)）。

IRQ 路径由
[`map_device_irqs_to_domain()`](../../xen/xen/arch/arm/device.c:111) 和
[`map_irq_to_domain()`](../../xen/xen/arch/arm/device.c:21) 处理：

- `irq_permit_access()` 授权；
- `vgic_reserve_virq()` 预留 virtual IRQ；
- `route_irq_to_guest()` 把硬件 IRQ 路由到 Dom0。

对于 `xen,passthrough` 设备，`own_device == false`：

- 不加入 Dom0 的 `dt_devices`；
- 不建立 MMIO mapping；
- 不路由 IRQ；
- 在 Dom0 DT 中把设备 `status` 设为 `"disabled"`。

### 7.7 从 Dom0 Device Tree 中隐藏 Xen 管理的 SMMU

[`write_properties()`](../../xen/xen/arch/arm/domain_build.c:475) 检测设备
是否引用 Xen 已识别的 IOMMU。如果是，则从 Dom0 DT 中删除：

```text
iommus
iommu-map
iommu-map-mask
```

同时，`handle_node()` 跳过 `DEVICE_IOMMU` 节点本身。

因此，在正常 Xen-managed SMMU 路径中，Dom0 Linux 通常看不到物理 SMMU
binding。它不会再去初始化或重新编程 SMMU，设备 DMA 由 Xen 的 stage-2
配置保护。

这也解释了一个常见现象：Dom0 内可能看不到 Linux IOMMU group/VFIO 关系，
但设备 DMA 已经受到 Xen SMMU 约束。不要把“Dom0 内没有 SMMU driver”误判为
“DMA 没有经过 SMMU”。

### 7.8 Dom0 直通的配置检查清单

在不运行实验的前提下，可以先从配置和生成结果做静态检查：

| 检查对象 | 期望 |
| --- | --- |
| host DT 的 SMMU 节点 | `status` 可用，`compatible` 匹配 Xen 支持的 SMMUv2/v3 |
| host DT 的设备身份 | 平台设备有正确的 `iommus`，PCI bridge 有正确的 `iommu-map` |
| Dom0 设备标记 | 目标设备不应带 `xen,passthrough` |
| Dom0 生成的 DT | 普通设备为 `"okay"`，带有正确 MMIO 和 IRQ |
| Xen-managed SMMU | Dom0 DT 中通常没有 SMMU 节点和 `iommus`/`iommu-map` 属性 |
| SMMU 候选 | Xen 启动日志应表明 IOMMU 已启用，并识别到对应 master |
| 后续运行证据 | 需要确认实际 driver probe、DMA 和 SMMU fault 情况，不能只看 DT |

如果目标设备的 host DT 没有 `iommus`/`iommu-map`，Xen 不会凭空知道该设备
经过哪个 SMMU。此时即使平台上存在 SMMU，也可能只走无 IOMMU 的设备所有权
路径。

如果目标设备仍带 `xen,passthrough`，它会被改写成 Dom0 DT 中的
`status = "disabled"`，不能被解释为“Dom0 直通已开启”。

### 7.9 Xen-managed SMMU 与 Dom0-managed SMMU

本文描述的是 Xen 找到并初始化了物理 SMMU，然后抢占 SMMU 所有权的主路径。

如果 host DT 中的 SMMU 没有被 Xen driver 识别：

- `device_get_class(iommu_node)` 不会返回 `DEVICE_IOMMU`；
- `write_properties()` 不会按 Xen-managed IOMMU 规则删除 `iommus`;
- Dom0 可能看到 SMMU 节点并由 Linux SMMU driver 管理；
- 此时 Xen P2M 与物理 SMMU 的共享方式、Dom0 中 IOMMU group 和生产安全
  边界都会不同。

因此，看到“Dom0 中存在 SMMU 或 IOMMU group”时，应先确认所有权模型，不能
直接套用本文的 Xen-managed stage-2 调用链。

## 8. SMMU attach 后如何形成可用的 DMA 路径

### 8.1 SMMUv3 attach

[`arm_smmu_assign_dev()`](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:2685)
的主要流程：

1. 对 PCI 设备更新 `pdev->domain` 和 domain 的 PCI 设备链表；
2. 在当前 Xen domain 的 `contexts` 链表中，按 SMMU 查找已有 IOMMU
   translation domain；
3. 没有则分配 `arm_smmu_domain` 并挂入链表；
4. 调用 [`arm_smmu_attach_dev()`](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:1427)；
5. 第一次 attach 时调用 `arm_smmu_domain_finalise()`；
6. 根据目标 Xen domain 的 P2M root 生成 stage-2 `VTTBR/VTCR`；
7. 为设备的所有 SID 安装 STE；
8. 把 master 加入该 SMMU domain 的设备链表；
9. 必要时配置 ATS。

attach 后，设备的 StreamID 在 Stream Table 中从 bypass/fault 切换为
`S2_TRANS`，其翻译根就是 Dom0 的 P2M。

### 8.2 SMMUv2 attach

[`arm_smmu_assign_dev()`](../../xen/xen/drivers/passthrough/arm/smmu.c:2751)
采用类似结构：

1. 为 Xen domain 和当前 SMMU 选择或创建 context bank 容器；
2. `arm_smmu_attach_dev()` 验证设备；
3. 初始化 context bank 的 stage-2 配置；
4. 把 `domain->arch.p2m.root` 写入 TTBR0；
5. 设置 CBAR 为 stage-2 translation 和 VMID；
6. SMR/S2CR 把设备的 StreamID 指向该 context bank；
7. 增加 IOMMU domain 的引用计数。

SMMUv2 的日志中可以看到类似：

```text
arm-smmu ...: d0: p2maddr 0x...
```

该 `p2maddr` 就是设备与 CPU 共享 stage-2 页表的直接证据。

### 8.3 运行时 DMA

以 Dom0 网卡为例：

```text
Dom0 net driver
  -> writes DMA descriptor with IPA
  -> device emits memory read/write
  -> StreamID selects SMMU context
  -> context stage-2 walks P2M
  -> HPA
```

设备看到的是 Dom0 的 IPA 空间。Dom0 内核管理自己的 CPU 页表、DMA buffer
和 cache 操作；Xen 管理 P2M 和物理 SMMU 上下文。

## 9. P2M 更新与 IOTLB 失效

### 9.1 共享页表意味着一次更新影响两套翻译缓存

CPU 和 SMMU 共享 P2M，因此修改 P2M 后，除了 CPU stage-2 TLB，还必须处理
SMMU 的 translation cache。ARM P2M 写入路径在
[`__p2m_set_entry()`](../../xen/xen/arch/arm/mmu/p2m.c:955) 中检测：

```c
if ( is_iommu_enabled(p2m->domain) &&
     (lpae_is_valid(orig_pte) || lpae_is_valid(*entry)) )
    iommu_iotlb_flush(...);
```

具体位置见
[ARM P2M flush 路径](../../xen/xen/arch/arm/mmu/p2m.c:1139)。

这覆盖：

- 新增 translation；
- 修改 translation 或权限；
- 删除 translation。

不同 SMMU driver 的 `iotlb_flush` 最终会发出 SMMUv2 TLB invalidate 或
SMMUv3 `CMDQ_OP_TLBI_*`。

### 9.2 Coherent table walk

Xen 会检查所有 SMMU 是否支持 coherent page-table walk，并设置
`IOMMU_FEAT_COHERENT_WALK`。P2M 在非 coherent table walk 平台上会对页表页
做必要的 cache clean，使 SMMU 能看到新表项。

即使 page-table walk coherent，设备 DMA 数据本身的 cache coherence 仍由
设备和平台决定。IOMMU coherent walk 不等于设备 master 与 CPU cache 一致。

### 9.3 页表与生命周期顺序

ARM domain 销毁时先调用 `iommu_domain_destroy()`，再调用
`p2m_final_teardown()`
（[domain.c](../../xen/xen/arch/arm/domain.c:824)）。原因是 SMMU context
仍然引用 P2M 根；必须先拆除设备/context，再释放 P2M。

## 10. 无 IOMMU、DomU 和 Dom0less 的区别

### 10.1 无 IOMMU

如果系统没有可用 SMMU，或 SMMU 节点被禁用：

- `iommu_setup()` 可能返回 `-ENODEV`，Xen 继续以
  `iommu_enabled == false` 启动；
- Dom0 仍可通过 `handle_device()` 获得普通设备的 MMIO 和 IRQ；
- 设备 DMA 没有 Xen-managed IOMMU 保护；
- 这条路径只适合可信 Dom0 或平台已有其他隔离机制的场景。

Xen 文档也明确警告，无 IOMMU assignment 只应把设备交给可信 guest，并需要
额外安全机制
（[passthrough-noiommu.txt](../../xen/docs/misc/arm/passthrough-noiommu.txt:3)）。

### 10.2 把设备给 DomU

典型顺序是：

1. 在 host DT 中给设备加 `xen,passthrough`；
2. Xen 启动时不让 Dom0 获得该设备的 MMIO/IRQ；
3. Dom0 中的 toolstack 使用设备树片段和 `dtdev`/`iomem`/`irqs`
   描述 guest 视图；
4. `XEN_DOMCTL_assign_device` 进入
   [`iommu_do_dt_domctl()`](../../xen/xen/drivers/passthrough/device_tree.c:321)；
5. Xen 调用 `iommu_add_dt_device()`，再调用
   `iommu_assign_dt_device(target_domain, dev)`；
6. SMMU 重新 attach 到目标 domain 的 P2M。

核心 SMMU attach 函数与 Dom0 路径相同，区别只是目标 `struct domain` 不同。

文档示例见
[ARM passthrough.txt](../../xen/docs/misc/arm/passthrough.txt:1)。

### 10.3 Dom0less guest

没有 Dom0 toolstack 时，guest 的 partial Device Tree 使用：

- `xen,path`：host DT 中对应设备路径；
- `xen,reg`：物理 MMIO 区间和 guest MMIO 地址；
- `xen,force-assign-without-iommu`：允许无 IOMMU assignment。

`handle_passthrough_prop()` 会调用 `iommu_add_dt_device()` 和
`iommu_assign_dt_device()`
（[dom0less-build.c](../../xen/xen/common/device-tree/dom0less-build.c:230)）。

无 IOMMU 的强制路径会放弃 DMA 边界，文档明确标注会降低平台安全性，不应
把它当作“普通直通配置”。

## 11. 常见误区

### 误区一：Dom0 直通一定要在 Dom0 里用 VFIO

错。这里讨论的是 Xen hypervisor 把物理设备交给 hardware domain。Dom0
直接使用原生驱动，不需要 Linux VFIO，也不需要 QEMU `vfio-pci`。

VFIO 是 Linux/KVM/QEMU 场景中把设备从 Host kernel 驱动转交 userspace VMM
的框架。Xen ARM 的 Dom0 直通主路径在 Xen 内完成 SMMU attach。

### 误区二：给设备加 `xen,passthrough` 就会直通给 Dom0

错。它会把设备从 Dom0 隐藏，典型用途是后续分配给 DomU。Dom0 普通设备默认
就被 Dom0 拥有。

### 误区三：Dom0 没有 SMMU 节点，所以 DMA 不经过 SMMU

在当前 Xen-managed SMMU 路径中，Xen 故意从 Dom0 DT 删除 SMMU binding 和
SMMU 节点。Dom0 看不到 SMMU 是设计结果，不代表硬件 SMMU 没有参与 DMA。

### 误区四：设备 attach 到 SMMU 后只能访问自己的 BAR

错。stage-2 是 domain 地址空间。Dom0 的设备可以翻译到 Dom0 P2M 中被允许的
内存和 MMIO，不只是设备自己的 BAR。若要设备级最小权限，需要额外的平台或
设备域设计。

### 误区五：IOMMU 能替代设备 reset 和 GPU 内部隔离

错。SMMU 负责 DMA 地址翻译和访问控制。设备 firmware、GPU 内部 context、
VRAM 分区、reset、ATS/PASID 状态和 cache coherence 是不同层的问题。

### 误区六：`dom0-iommu=passthrough` 在 ARM 上会关闭 SMMU stage-2

在本基线中不能这样理解。通用代码解析并打印该模式，但 ARM 的
`arch_iommu_hwdom_init()` 只禁用部分 unsupported 选项；SMMUv2/v3 代码仍
明确使用 stage-2。该参数的历史语义和实际作用需要结合目标 Xen 版本继续核对，
不能用它替代对 SMMU attach 代码的检查。

## 12. 一条完整调用链

### 12.1 Dom0 启动路径

```text
start_xen()
  -> iommu_setup()
    -> iommu_hardware_setup()
      -> device_init(DEVICE_IOMMU)
        -> arm_smmu_dt_init()
          -> arm_smmu_device_probe()
          -> iommu_set_ops(&arm_smmu_iommu_ops)
    -> setup_virt_paging()
  -> create_dom0()
    -> domain_create()
      -> arch_domain_create()
        -> iommu_domain_init()
          -> arm_smmu_iommu_*_domain_init()
        -> p2m_init()
          -> p2m_alloc_table()
    -> construct_dom0()
      -> construct_hwdom()
        -> iommu_hwdom_init()
        -> allocate_memory() / allocate_memory_11()
        -> prepare_dtb_hwdom()
          -> handle_node()
            -> handle_device()
              -> iommu_add_dt_device()
                -> iommu_dt_xlate()
                -> ops->add_device()
              -> iommu_assign_dt_device()
                -> ops->assign_device()
                  -> arm_smmu_assign_dev()
                    -> arm_smmu_attach_dev()
                    -> arm_smmu_domain_finalise_s2()
              -> map_device_irqs_to_domain()
              -> map_range_to_domain()
        -> construct_domain()
```

### 12.2 DMA 时

```text
Dom0 device driver
  -> device DMA transaction with StreamID
  -> SMMUv2 SMR/S2CR/context bank
     or SMMUv3 Stream Table/STE
  -> Dom0 P2M root
  -> HPA
```

### 12.3 P2M 修改时

```text
p2m_set_entry()
  -> 更新/删除 P2M entry
  -> iommu_iotlb_flush()
  -> SMMUv2 TLB invalidate
     or SMMUv3 CMDQ TLBI + SYNC
```

## 13. 建议的源码阅读顺序

第一次阅读不要直接跳进 SMMUv2/SMMUv3 大文件。建议按以下顺序：

1. [ARM passthrough 用户文档](../../xen/docs/misc/arm/passthrough.txt:1)：
   先建立 `xen,passthrough` 和 guest partial DT 的配置语义。
2. [ARM device passthrough 文档](../../xen/docs/misc/arm/device-tree/passthrough.txt:1)：
   明确该属性会把设备从 Dom0 隐藏。
3. [`xen/arch/arm/setup.c`](../../xen/xen/arch/arm/setup.c:281)：
   看 IOMMU、P2M、Dom0 的全局启动顺序。
4. [`xen/drivers/passthrough/iommu.c`](../../xen/xen/drivers/passthrough/iommu.c:196)：
   看通用 domain lifecycle、`iommu_map()` 和 `iommu_hwdom_init()`。
5. [`xen/arch/arm/domain.c`](../../xen/xen/arch/arm/domain.c:665)：
   确认 IOMMU domain init 在 P2M 前。
6. [`xen/arch/arm/domain_build.c`](../../xen/xen/arch/arm/domain_build.c:1470)：
   看 Dom0 DT 如何遍历和过滤设备。
7. [`xen/arch/arm/device.c`](../../xen/xen/arch/arm/device.c:248)：
   看 Dom0 设备所有权、MMIO 和 IRQ。
8. [`xen/drivers/passthrough/device_tree.c`](../../xen/xen/drivers/passthrough/device_tree.c:29)：
   看 `add_device` 和 `assign_device` 的框架边界。
9. [`xen/drivers/passthrough/arm/smmu-v3.c`](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:1112)：
   先看 SMMUv3 domain 分配、stage-2 finalise、attach 和 STE。
10. [`xen/drivers/passthrough/arm/smmu.c`](../../xen/xen/drivers/passthrough/arm/smmu.c:1157)：
    再对照 SMMUv2 context bank、SMR/S2CR。
11. [`xen/arch/arm/mmu/p2m.c`](../../xen/xen/arch/arm/mmu/p2m.c:1427)：
    最后看根页表生命周期、P2M entry 更新和 IOTLB flush。

## 14. 关键源码索引

| 主题 | 源码 |
| --- | --- |
| 全局 IOMMU lifecycle | [`iommu.c`](../../xen/xen/drivers/passthrough/iommu.c:196) |
| ARM IOMMU 后端发现 | [`arm/iommu.c`](../../xen/xen/drivers/passthrough/arm/iommu.c:51) |
| DT fwspec/add/assign | [`device_tree.c`](../../xen/xen/drivers/passthrough/device_tree.c:29) |
| Dom0 DT 构建 | [`domain_build.c`](../../xen/xen/arch/arm/domain_build.c:1470) |
| Dom0 设备 MMIO/IRQ | [`device.c`](../../xen/xen/arch/arm/device.c:248) |
| ARM P2M 初始化 | [`mmu/p2m.c`](../../xen/xen/arch/arm/mmu/p2m.c:1427) |
| ARM P2M 与 IOMMU flush | [`mmu/p2m.c`](../../xen/xen/arch/arm/mmu/p2m.c:1139) |
| SMMUv2 context bank | [`smmu.c`](../../xen/xen/drivers/passthrough/arm/smmu.c:1157) |
| SMMUv2 domain assign | [`smmu.c`](../../xen/xen/drivers/passthrough/arm/smmu.c:2751) |
| SMMUv3 stage-2 finalise | [`smmu-v3.c`](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:1163) |
| SMMUv3 domain assign | [`smmu-v3.c`](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:2685) |
| Dom0less passthrough | [`dom0less-build.c`](../../xen/xen/common/device-tree/dom0less-build.c:230) |

## 15. 读完后应能回答的问题

1. 为什么 SMMU 必须用 StreamID 选择翻译上下文？
2. SMMUv2 的 SMR、S2CR 和 context bank 分别负责什么？
3. SMMUv3 的 Stream Table、STE、VTCR 和 VTTBR 分别负责什么？
4. 为什么 Xen 可以把 CPU P2M 直接作为 SMMU stage-2？
5. 为什么 Dom0 看不到 SMMU 节点，设备 DMA 仍可以被 SMMU 翻译？
6. `xen,passthrough` 为什么不是“Dom0 直通开关”？
7. `iommu_add_dt_device()` 和 `iommu_assign_dt_device()` 的职责边界是什么？
8. `iommu_assign_dt_device()` 为什么不负责建立 MMIO 映射？
9. SMMU attach 后，Dom0 设备为什么能访问 Dom0 的其他 RAM？
10. P2M entry 修改后为什么必须失效 SMMU TLB？
11. `xen,force-assign-without-iommu` 放弃了什么安全边界？
12. 为什么 IOMMU 隔离不能替代设备 reset 和 GPU 内部隔离？

## 16. 本轮边界

本文完成的是一份源码驱动的方案说明。以下内容尚未通过运行验证：

- 某款具体 SoC 的 SMMUv2/v3 兼容性；
- 某款 GPU 的 ACS、ATS/PASID、reset 和 firmware 行为；
- Dom0 中真实设备驱动的 DMA mask、cache 属性和 IOMMU 地址窗口；
- Device Tree 中 bus address、IPA 和 HPA 是否存在平台特定偏移；
- ACPI/IORT boot 路径和 Dom0 ACPI 设备直通的完整性；
- 故障注入、性能、热插拔、迁移和安全攻击面。

下一步应当在确定具体 ARM 平台、Xen 版本、SMMU 版本和目标设备后，先补齐
host DT、SMMU 能力和 Dom0 日志，再设计最小直通实验。
