# ARM SMMU 原理与 Linux/Xen 源码对照

## 1. 文档目的

本文用一条统一的硬件模型解释 ARM System MMU（SMMU），再把同一组概念逐项
对应到 Linux 和 Xen 的实现：

```text
设备如何标识自己？
  -> StreamID / Requester ID

设备标识如何找到翻译上下文？
  -> SMMUv2: SMR + S2CR + Context Bank
  -> SMMUv3: Stream Table + STE + CD/VTCR/VTTBR

上下文如何使用页表？
  -> stage-1 / stage-2
  -> LPAE / V7S page table

软件如何建立、修改和失效翻译？
  -> Linux: IOMMU core + DMA API + io-pgtable + SMMU driver
  -> Xen: Xen iommu framework + P2M + SMMU driver

设备如何产生 fault 和 page request？
  -> SMMUv2 context fault / SMMUv3 eventq + priq
```

本文只做静态源码分析，不启动 Linux、Xen 或 SMMU 实验。

文中的代码块均是从固定源码中截取的关键路径。为突出机制，会省略日志、
参数检查、错误回滚、锁实现和与当前问题无关的分支；省略处用 `...` 表示。

源码基线：

```text
Linux 6b4b543fa18757297ac756a6fe6e64f506b7844a
Xen 4.23-dev-454-gdf8d101a38
Xen commit df8d101a3854725fd3db3c52f074808e6b3cc539
```

关联文档：

- [ARM IOMMU 与 Xen Dom0 设备直通](README.md)
- [通用 IOMMU Device Tree binding](../../linux/Documentation/devicetree/bindings/iommu/iommu.txt:1)
- [ARM SMMU binding](../../linux/Documentation/devicetree/bindings/iommu/arm,smmu.yaml:1)
- [ARM SMMUv3 binding](../../linux/Documentation/devicetree/bindings/iommu/arm,smmu-v3.yaml:1)

### 1.1 ARM 官方 SMMU 规范

以下版本于 2026-11-10 通过 Arm Developer Documentation API 核验：

| 架构 | 规范标题 | 文档号/版本 | 发布日期 |
| --- | --- | --- | --- |
| SMMUv2 | *ARM System Memory Management Unit Architecture Specification - SMMU architecture version 2.0* | `ARM IHI 0062D.c` | 2016-06-30 |
| SMMUv3 | *Arm System Memory Management Unit Architecture Specification, v3* | `ARM IHI 0070 H.a` | 2026-03-06 |

SMMUv2 官方页面与 PDF：

- [Arm 官方文档页：IHI 0062 2.0 D.c](https://support.arm.com/documentation/ihi0062/dc/)
- [官方 PDF：`IHI0062D_c_system_mmu_architecture_specification.pdf`](https://documentation-service.arm.com/static/5f900d34f86e16515cdc08fb?token=)
- ARM Developer 入口：
  <https://developer.arm.com/documentation/ihi0062/dc/>

SMMUv3 官方页面与 PDF：

- [Arm 官方文档页：IHI 0070 H.a](https://support.arm.com/documentation/ihi0070/ha/)
- [官方 PDF：`IHI0070H_a-System_Memory_Management_Unit_Architecture_Specification.pdf`](https://documentation-service.arm.com/static/6af19af08dd1453dd1a3c5d2?token=)
- ARM Developer 入口：
  <https://developer.arm.com/documentation/ihi0070/ha/>

版本边界：

- `IHI 0062 D.c` 是当前 SMMUv2 2.0 规范的最新发布修订；
- `IHI 0070 H.a` 汇总 SMMUv3.0 到 SMMUv3.5，并包含 RME/RME DA
  相关扩展；
- 某个 SoC 标称“SMMUv3”不代表实现支持 H.a 的全部可选特性；应以
  `IDR0~IDR5` 等能力寄存器和 driver feature bits 为准；
- 本文不把 Arm PDF 复制进仓库，避免二进制文件版本漂移和版权边界混淆；
  上表的官方 PDF 链接是权威来源。

### 1.2 规范章节索引

SMMUv2 阅读重点：

| 规范章节 | 内容 | 本文对应 |
| --- | --- | --- |
| Chapter 1 Introduction | SMMU 架构、translation regimes、translation schemes、地址能力 | 第 2、3 节 |
| 2.3 Context determination | 如何由设备请求确定 stream mapping 和上下文 | 5.2-5.5 |
| 2.5 TLB operation | SMMU translation cache 和失效 | 第 16 节 |
| 2.6 Translation context | SMR/S2CR/context bank 的上下文关系 | 5.3-5.5 |
| 2.7 Translation and protection checks | stage-1/stage-2 访问权限 | 第 3、11 节 |
| Chapter 3 The Fault Model | context fault、global fault、fault interrupt | 5.6、6.6、7.5 |
| Chapter 5 Coherency and Cache Maintenance | page-table walk coherency、TLB/cache maintenance | 16.1-16.3 |
| Chapters 8-10 | SMMU address space、Global Register Space 0/1 | 5.1、6.1 |
| Chapters 14-15 | Extended Stream Matching、compressed StreamID indexing | 5.2-5.4 |
| Chapter 16 | stage-1 context bank 格式 | 5.5、11.1 |
| Chapter 17 | stage-2 context bank 格式 | 5.5、7.3、11.1 |

SMMUv3 阅读重点：

| 规范章节 | 内容 | 本文对应 |
| --- | --- | --- |
| Chapter 2 Introduction | SMMUv3.0-v3.5 feature history | 8.1、10.1 |
| 3.3 Stream table and translation procedure | Stream table lookup、StreamID 到 CD、config/translation lookup | 8.2-8.5 |
| 3.5 Command and Event queues | CMDQ、EVTQ、queue 可见性 | 8.6-8.8、9.1 |
| 3.9 PCIe, PASIDs, PRI, ATS | ATS/PASID/PRI 与 SMMU 的接口 | 8.8-8.9、9.7、10.7 |
| 3.11 Reset, Enable and initialization | SMMU reset、queue/table enable | 9.1、10.1 |
| 3.12 Fault models, recording and reporting | terminate/stall fault 模型 | 8.7-8.8、17 |
| 3.17 TLB tagging/VMID/ASID/invalidation | TLB 标签和广播失效 | 16.1-16.2 |
| 3.21 Structure access rules and update procedures | STE/CD 更新顺序及 sync | 9.5 |
| 4.3 Configuration structure invalidation | `CMD_CFGI_STE`、`CMD_CFGI_CD` | 8.6、9.5 |
| 4.4 TLB invalidation | stage-1/stage-2 TLBI | 10.6、16.1 |
| 4.5 ATS and PRI | `CMD_ATC_INV`、`CMD_PRI_RESP` | 9.7、10.7、16.2 |
| 4.7 Fault response and sync | `CMD_RESUME`、`CMD_STALL_TERM`、`CMD_SYNC` | 8.6-8.8 |
| 5.1 L1STD | 两级 Stream Table 的 L1 descriptor | 8.2、9.2 |
| 5.2 STE | Stream Table Entry 全字段 | 8.3-8.4、9.4 |
| 5.3 L1CD / 5.4 CD | stage-1/PASID context descriptor | 8.5、9.7 |
| Chapter 6 Memory map and registers | `IDR`、`CR0/CR1/CR2`、STRTAB/CMDQ/EVTQ/PRIQ 寄存器 | 9.1、10.1 |
| Chapter 7 Command/Event queue errors/faults | queue error 和 event fault 编码 | 8.7、17 |

### 1.3 规范对象与代码对象

| 规范对象 | Linux | Xen |
| --- | --- | --- |
| SMMU instance | `struct arm_smmu_device` | `struct arm_smmu_device` |
| SMMUv2 SMR | `smmu->smrs[]` | `smmu->smrs[]` |
| SMMUv2 S2CR | `smmu->s2crs[]` | `smmu->s2crs[]` |
| SMMUv2 context bank | `struct arm_smmu_domain` + `smmu->cbs[]` | `struct arm_smmu_domain` + context bank 寄存器 |
| SMMUv3 Stream Table | `smmu->strtab_cfg` | `smmu->strtab_cfg` |
| STE | `struct arm_smmu_ste` | `__le64 *step` / STE dwords |
| CD | `struct arm_smmu_cd` | Xen 主路径不建立完整 S1 CD 地址空间 |
| Stage-2 page table | `io-pgtable` + `ARM_64_LPAE_S2` | Xen domain P2M |
| CMDQ | `struct arm_smmu_cmdq` | command queue software state |
| EVTQ/PRIQ | `struct arm_smmu_queue` | event/pri queue software state |

阅读规范时要牢记：规范定义“硬件允许怎样实现”，驱动代码定义“这个操作系统
如何使用其中一部分能力”。两者不能直接画等号。

## 2. 一句话理解 SMMU

CPU MMU 翻译“CPU 发出的地址”；SMMU 翻译“设备发出的 DMA 地址”。

```text
CPU core
  -> CPU VA
  -> CPU MMU
  -> PA

Device
  -> DMA address
  -> SMMU
  -> PA
```

SMMU 的职责不只有地址翻译，还包括：

- 用设备身份隔离不同设备的 DMA；
- 把设备访问限制在允许的页和权限内；
- 对非法访问产生 fault；
- 维护 TLB、ATS translation cache 等翻译缓存的一致性；
- 为 PCIe ATS、PRI、PASID 等功能提供硬件基础。

SMMU 不负责：

- 设备固件和命令处理器安全；
- GPU/加速器内部 context 隔离；
- 设备 reset；
- CPU cache 与设备 DMA 数据的自动一致性；
- 没有经过 SMMU 的内部 MMIO、VRAM 或本地内存访问。

## 3. 地址空间和两个 translation stage

### 3.1 Stage-1

Stage-1 把软件可见的虚拟地址翻译为中间物理地址：

```text
VA -> IPA
```

对 ARM CPU，这类似 EL0/EL1 的普通页表翻译。对 IOMMU：

- Linux DMA API 常使用 IOMMU stage-1；
- 设备驱动拿到的 DMA address 是 IOVA；
- IOVA 通过 stage-1 翻译到物理页；
- PASID/SVA 可以为同一设备提供多个 stage-1 地址空间。

### 3.2 Stage-2

Stage-2 把 guest/hypervisor 的 IPA 翻译为真实物理地址：

```text
IPA -> PA
```

对 CPU：

```text
Guest VA
  -> stage-1 page table
  -> Guest IPA
  -> stage-2 / P2M
  -> Host PA
```

对 passthrough 设备：

```text
Device DMA address
  -> SMMU stage-2
  -> Host PA
```

### 3.3 Nested translation

同时启用两层时：

```text
Device IOVA
  -> SMMU stage-1 (GIOVA -> GPA/IPA)
  -> SMMU stage-2 (GPA/IPA -> HPA)
```

Linux SMMUv3 支持 stage-1、stage-2 和一定的 nested/IOMMUFD 组合；
当前 Xen ARM passthrough 主路径只使用 stage-2。

## 4. 设备身份：StreamID 与 Requester ID

### 4.1 为什么不能只看 DMA 地址

两个设备可能向总线发出相同的 DMA 地址，但它们应该访问不同的页表：

```text
GPU    -> IOVA 0x1000
NVMe   -> IOVA 0x1000
```

如果 SMMU 只根据地址查找页表，无法区分请求来源。因此 SMMU 先使用设备身份
选择 translation context，再做地址翻译。

### 4.2 平台设备

Device Tree 通常写成：

```dts
smmu: iommu@40000000 {
    compatible = "arm,smmu-v3";
    #iommu-cells = <1>;
};

device@50000000 {
    reg = <0x0 0x50000000 0x0 0x1000>;
    iommus = <&smmu 0x1234>;
};
```

对 SMMUv2/v3，`0x1234` 是设备 master interface 提交给 SMMU 的
StreamID。

Linux 解析设备 `iommus` 的关键代码
（[of_iommu.c](../../linux/drivers/iommu/of_iommu.c:60)）：

```c
static int of_iommu_configure_dev(struct device_node *master_np,
                                  struct device *dev)
{
    struct of_phandle_args iommu_spec;
    int err = -ENODEV, idx = 0;

    while (!of_parse_phandle_with_args(master_np, "iommus",
                                       "#iommu-cells",
                                       idx, &iommu_spec)) {
        err = of_iommu_xlate(dev, &iommu_spec);
        of_node_put(iommu_spec.np);
        idx++;
        if (err)
            break;
    }

    return err;
}
```

`of_iommu_xlate()` 把 IOMMU 设备与 ID 存入 fwspec，再调用具体 driver：

```c
ret = iommu_fwspec_init(dev, of_fwnode_handle(iommu_spec->np));
...
ops = iommu_ops_from_fwnode(&iommu_spec->np->fwnode);
ret = ops->of_xlate(dev, iommu_spec);
```

SMMUv2 的 `of_xlate` 将 `iommus` 的 ID/mask 编码进 fwspec
（[arm-smmu.c](../../linux/drivers/iommu/arm/arm-smmu/arm-smmu.c:1587)）：

```c
static int arm_smmu_of_xlate(struct device *dev,
                             const struct of_phandle_args *args)
{
    u32 mask, fwid = 0;

    if (args->args_count > 0)
        fwid |= FIELD_PREP(ARM_SMMU_SMR_ID, args->args[0]);

    if (args->args_count > 1)
        fwid |= FIELD_PREP(ARM_SMMU_SMR_MASK, args->args[1]);
    else if (!of_property_read_u32(args->np, "stream-match-mask", &mask))
        fwid |= FIELD_PREP(ARM_SMMU_SMR_MASK, mask);

    return iommu_fwspec_add_ids(dev, &fwid, 1);
}
```

### 4.3 PCI 设备

PCIe transaction 带 Requester ID，通常是 BDF：

```text
Requester ID = Bus/Device/Function
```

PCI host bridge 用 `iommu-map` 将 RID/BDF 映射为 SMMU StreamID：

```text
PCI Requester ID
  -> host bridge iommu-map
  -> StreamID
  -> SMMU
```

这就解释了为什么 PCI endpoint 本身通常没有静态 `iommus`：
拓扑和映射属于 host bridge。

### 4.4 Linux 和 Xen 的共同数据结构

两者都使用 `struct iommu_fwspec` 保存设备固件描述：

```text
fwspec->iommu_fwnode / iommu_dev
fwspec->num_ids
fwspec->ids[]
```

Linux 中设备通过 `of_iommu_configure()` 解析 `iommus` 或 PCI alias，
再调用 SMMU driver 的 `of_xlate`：

- [`of_iommu_xlate()`](../../linux/drivers/iommu/of_iommu.c:22)
- [`of_iommu_configure()`](../../linux/drivers/iommu/of_iommu.c:114)
- SMMUv2 [`of_xlate`](../../linux/drivers/iommu/arm/arm-smmu/arm-smmu.c:1587)
- SMMUv3 [`of_xlate`](../../linux/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c:4543)

Xen 在 `iommu_add_dt_device()` 中做同职责的工作：

- [`iommu_dt_xlate()`](../../xen/xen/drivers/passthrough/device_tree.c:140)
- [`iommu_add_dt_device()`](../../xen/xen/drivers/passthrough/device_tree.c:258)
- [`iommu_fwspec_init()`](../../xen/xen/drivers/passthrough/arm/iommu_fwspec.c:32)

共同逻辑是：

```text
固件描述
  -> 找到 SMMU
  -> 建立 device fwspec
  -> driver 将 ID 转成硬件 SID/stream match 格式
  -> driver 登记 master
```

Xen 完成同一件事的核心代码
（[device_tree.c](../../xen/xen/drivers/passthrough/device_tree.c:140)）：

```c
static int iommu_dt_xlate(struct device *dev,
                          const struct dt_phandle_args *iommu_spec,
                          const struct iommu_ops *ops)
{
    int rc;
    ...
    rc = iommu_fwspec_init(dev, &iommu_spec->np->dev);
    if ( rc )
        return rc;

    return ops->dt_xlate(dev, iommu_spec);
}
```

Xen 遍历所有 `iommus`，然后调用 driver 的 `add_device`
（[device_tree.c](../../xen/xen/drivers/passthrough/device_tree.c:258)）：

```c
while ( !dt_parse_phandle_with_args(np, "iommus", "#iommu-cells",
                                    index, &iommu_spec) )
{
    rc = iommu_dt_xlate(dev, &iommu_spec, ops);
    if ( rc )
        break;
    index++;
}

if ( !rc )
    rc = ops->add_device(0, dev);
```

## 5. SMMUv2 详细原理

### 5.1 硬件组成

SMMUv2 主要由以下部分组成：

```text
Global registers
  +-----------------------+
  | SMR[n]                |  Stream Match Register
  | S2CR[n]               |  Stream-to-Context Register
  | fault registers       |
  +-----------------------+

Context Bank[n]
  +-----------------------+
  | CBAR                  |  translation type / VMID / IRQ
  | TTBR0/TTBR1           |  page table root
  | TCR/TTBCR             |  translation configuration
  | MAIR                  |  memory attributes
  | SCTLR                 |  context enable
  | FSR/FAR               |  fault info
  +-----------------------+
```

### 5.2 Stream indexing

如果 SMMU 没有可编程 stream matching，SID 可以直接作为 S2CR 下标：

```text
SID = 0x25
  -> S2CR[0x25]
```

这种方式简单，但流 ID 空间必须足够密集且硬件资源允许。

### 5.3 Stream matching

支持 matching 时：

```text
StreamID
  -> 与每个 SMR 的 ID/MASK 比较
  -> 命中 SMR[n]
  -> 使用 S2CR[n]
```

SMR 大致包含：

```text
ID
MASK
VALID
```

例如：

```text
SMR.ID   = 0x1000
SMR.MASK = 0x00ff
```

可匹配一组具有相同高位的 StreamID。MASK 让一个 SMR 覆盖多个设备 ID，
但也带来别名和冲突问题。

### 5.4 S2CR

S2CR 决定命中 StreamID 后的行为：

| 类型 | 行为 |
| --- | --- |
| `TRANS` | 进入指定 context bank 做地址翻译 |
| `BYPASS` | 不做 stage-2 翻译，地址直通 |
| `FAULT` | 直接产生 fault |

S2CR 的 `CBNDX` 指向 Context Bank。真正是否进行 stage-1、stage-2 或
nested 翻译，再由 CBAR 和 context bank 配置决定。

Linux 写 SMR 和 S2CR 的代码直接对应上述硬件字段
（[arm-smmu.c](../../linux/drivers/iommu/arm/arm-smmu/arm-smmu.c:950)）：

```c
static void arm_smmu_write_smr(struct arm_smmu_device *smmu, int idx)
{
    struct arm_smmu_smr *smr = smmu->smrs + idx;
    u32 reg = FIELD_PREP(ARM_SMMU_SMR_ID, smr->id) |
              FIELD_PREP(ARM_SMMU_SMR_MASK, smr->mask);

    if (!(smmu->features & ARM_SMMU_FEAT_EXIDS) && smr->valid)
        reg |= ARM_SMMU_SMR_VALID;
    arm_smmu_gr0_write(smmu, ARM_SMMU_GR0_SMR(idx), reg);
}

static void arm_smmu_write_s2cr(struct arm_smmu_device *smmu, int idx)
{
    struct arm_smmu_s2cr *s2cr = smmu->s2crs + idx;
    u32 reg;

    reg = FIELD_PREP(ARM_SMMU_S2CR_TYPE, s2cr->type) |
          FIELD_PREP(ARM_SMMU_S2CR_CBNDX, s2cr->cbndx) |
          FIELD_PREP(ARM_SMMU_S2CR_PRIVCFG, s2cr->privcfg);
    arm_smmu_gr0_write(smmu, ARM_SMMU_GR0_S2CR(idx), reg);
}
```

这段代码可以直接映射到硬件：

```text
smr->id / smr->mask  -> SMR.ID / SMR.MASK
smr->valid           -> SMR.VALID
s2cr->type           -> TRANS / BYPASS / FAULT
s2cr->cbndx          -> Context Bank index
```

### 5.5 Context Bank

Context Bank 保存某个翻译上下文：

- `CBAR`：stage-1/stage-2 翻译类型；
- `TTBR0/TTBR1`：页表根；
- `TCR/TTBCR`：地址宽度、页大小、cache/shareability；
- `ASID`：stage-1 地址空间标识；
- `VMID`：stage-2 地址空间标识；
- `SCTLR`：是否启用 translation；
- fault 寄存器。

stage-2 context 的翻译链：

```text
StreamID
  -> SMR/S2CR
  -> Context Bank
  -> TTBR0 points to stage-2 table
  -> IPA -> PA
```

### 5.6 SMMUv2 fault

非法 DMA 或页表错误通常表现为：

- context bank fault；
- global fault；
- S2CR 配置为 FAULT 的直接 fault。

context fault 通常可读到：

- fault address；
- fault status；
- fault syndrome；
- 产生 fault 的 SID；
- read/write 方向。

Linux 和 Xen 都从 context bank 寄存器读取这些信息。

## 6. SMMUv2 的 Linux 源码对应

### 6.1 探测 SMMU

Linux 入口：

- [`arm_smmu_device_probe()`](../../linux/drivers/iommu/arm/arm-smmu/arm-smmu.c:2118)
- [`arm_smmu_device_cfg_probe()`](../../linux/drivers/iommu/arm/arm-smmu/arm-smmu.c:1739)

`cfg_probe` 从 ID 寄存器读取：

- SMMUv1/V2；
- stage-1/stage-2/nested 支持；
- stream indexing 或 stream matching；
- SMR 数量；
- context bank 数量；
- stage-2-only context bank 数量；
- IPA/VA/PA 位宽；
- 支持页大小；
- coherent table walk；
- 16-bit VMID 等能力。

硬件能力如何变成软件 feature：

```c
id = arm_smmu_gr0_read(smmu, ARM_SMMU_GR0_ID0);

if (id & ARM_SMMU_ID0_S1TS)
    smmu->features |= ARM_SMMU_FEAT_TRANS_S1;
if (id & ARM_SMMU_ID0_S2TS)
    smmu->features |= ARM_SMMU_FEAT_TRANS_S2;
if (id & ARM_SMMU_ID0_NTS)
    smmu->features |= ARM_SMMU_FEAT_TRANS_NESTED;

if (id & ARM_SMMU_ID0_SMS) {
    smmu->features |= ARM_SMMU_FEAT_STREAM_MATCH;
    size = FIELD_GET(ARM_SMMU_ID0_NUMSMRG, id);
    smmu->smrs = devm_kcalloc(smmu->dev, size,
                              sizeof(*smmu->smrs), GFP_KERNEL);
}

id = arm_smmu_gr0_read(smmu, ARM_SMMU_GR0_ID1);
smmu->num_s2_context_banks = FIELD_GET(ARM_SMMU_ID1_NUMS2CB, id);
smmu->num_context_banks = FIELD_GET(ARM_SMMU_ID1_NUMCB, id);

id = arm_smmu_gr0_read(smmu, ARM_SMMU_GR0_ID2);
smmu->ipa_size = arm_smmu_id_size_to_bits(FIELD_GET(ARM_SMMU_ID2_IAS, id));
smmu->pa_size  = arm_smmu_id_size_to_bits(FIELD_GET(ARM_SMMU_ID2_OAS, id));
```

`arm_smmu_device_probe()` 随后：

1. 映射 MMIO；
2. 获取 IRQ；
3. 初始化 SMR/S2CR 软件状态；
4. 初始化 context bank 软件状态；
5. reset SMMU；
6. 把 `arm_smmu_ops` 注册到 Linux IOMMU core。

### 6.2 登记设备 StreamID

Linux 通用 IOMMU core 探测设备后调用
[`arm_smmu_probe_device()`](../../linux/drivers/iommu/arm/arm-smmu/arm-smmu.c:1431)。

它做：

```text
fwspec.ids
  -> 检查 SID/MASK 是否在硬件范围内
  -> 分配 arm_smmu_master_cfg
  -> arm_smmu_master_alloc_smes()
  -> 选择或复用 SMR/S2CR 条目
```

[`arm_smmu_master_alloc_smes()`](../../linux/drivers/iommu/arm/arm-smmu/arm-smmu.c:1082)
直接对应硬件：

```text
StreamID + mask
  -> arm_smmu_find_sme()
  -> SMR.ID / SMR.MASK
  -> S2CR.type / S2CR.cbndx
  -> 写硬件寄存器
```

对应 Linux 源码节选：

```c
static int arm_smmu_master_alloc_smes(struct device *dev)
{
    struct iommu_fwspec *fwspec = dev_iommu_fwspec_get(dev);
    struct arm_smmu_master_cfg *cfg = dev_iommu_priv_get(dev);
    struct arm_smmu_device *smmu = cfg->smmu;
    int i, idx, ret;
    ...

    for_each_cfg_sme(cfg, fwspec, i, idx) {
        u16 sid = FIELD_GET(ARM_SMMU_SMR_ID, fwspec->ids[i]);
        u16 mask = FIELD_GET(ARM_SMMU_SMR_MASK, fwspec->ids[i]);

        ...
        ret = arm_smmu_find_sme(smmu, sid, mask);
        if (ret < 0)
            goto out_err;

        idx = ret;
        if (smmu->smrs && smmu->s2crs[idx].count == 0) {
            smmu->smrs[idx].id = sid;
            smmu->smrs[idx].mask = mask;
            smmu->smrs[idx].valid = true;
        }
        smmu->s2crs[idx].count++;
        cfg->smendx[i] = (s16)idx;
    }

    for_each_cfg_sme(cfg, fwspec, i, idx)
        arm_smmu_write_sme(smmu, idx);
    ...
}
```

`arm_smmu_find_sme()` 同时负责冲突检查：如果两个设备的匹配范围重叠，
就只能复用兼容条目或拒绝配置。

### 6.3 分配和初始化 domain

[`arm_smmu_domain_alloc_paging()`](../../linux/drivers/iommu/arm/arm-smmu/arm-smmu.c:916)
先分配 `arm_smmu_domain`，但此时还不能初始化页表，因为尚未 attach 到具体
SMMU/device。

真正 attach 时调用
[`arm_smmu_init_domain_context()`](../../linux/drivers/iommu/arm/arm-smmu/arm-smmu.c:675)：

1. 默认选择 stage-1；若硬件没有 stage-1，则回退到 stage-2；
2. 选择 AArch64 LPAE 或 AArch32 long/short descriptor；
3. 分配 context bank；
4. 选择 VMID/ASID；
5. 调用 `alloc_io_pgtable_ops()` 建立 IOMMU 页表；
6. 把页表配置转成 CBAR/TCR/TTBR/MAIR；
7. 写 context bank；
8. 注册 context fault IRQ；
9. 发布 `smmu_domain->pgtbl_ops`。

Linux 选择 stage、context bank 和页表格式的核心代码：

```c
if (!(smmu->features & ARM_SMMU_FEAT_TRANS_S1))
    smmu_domain->stage = ARM_SMMU_DOMAIN_S2;
if (!(smmu->features & ARM_SMMU_FEAT_TRANS_S2))
    smmu_domain->stage = ARM_SMMU_DOMAIN_S1;

switch (smmu_domain->stage) {
case ARM_SMMU_DOMAIN_S1:
    cfg->cbar = CBAR_TYPE_S1_TRANS_S2_BYPASS;
    start = smmu->num_s2_context_banks;
    fmt = ARM_64_LPAE_S1;   /* AArch32 时为 ARM_32_LPAE_S1 */
    break;
case ARM_SMMU_DOMAIN_S2:
    cfg->cbar = CBAR_TYPE_S2_TRANS;
    start = 0;
    fmt = ARM_64_LPAE_S2;   /* AArch32 时为 ARM_32_LPAE_S2 */
    break;
}

ret = arm_smmu_alloc_context_bank(smmu_domain, smmu, dev, start);
cfg->cbndx = ret;
if (smmu_domain->stage == ARM_SMMU_DOMAIN_S2)
    cfg->vmid = cfg->cbndx + 1;
else
    cfg->asid = cfg->cbndx;

pgtbl_cfg = (struct io_pgtable_cfg) {
    .pgsize_bitmap = smmu->pgsize_bitmap,
    .ias           = ias,
    .oas           = oas,
    .coherent_walk = smmu->features & ARM_SMMU_FEAT_COHERENT_WALK,
    .tlb           = smmu_domain->flush_ops,
    .iommu_dev     = smmu->dev,
};

pgtbl_ops = alloc_io_pgtable_ops(fmt, &pgtbl_cfg, smmu_domain);
arm_smmu_init_context_bank(smmu_domain, &pgtbl_cfg);
arm_smmu_write_context_bank(smmu, cfg->cbndx);
smmu_domain->pgtbl_ops = pgtbl_ops;
```

写入 context bank 时，stage-1 和 stage-2 的关键差异是 TTBR/TCR 的来源
（[arm-smmu.c](../../linux/drivers/iommu/arm/arm-smmu/arm-smmu.c:519)）：

```c
stage1 = cfg->cbar != CBAR_TYPE_S2_TRANS;

if (stage1) {
    cb->tcr[0] = arm_smmu_lpae_tcr(pgtbl_cfg);
    cb->ttbr[0] = FIELD_PREP(ARM_SMMU_TTBRn_ASID, cfg->asid) |
                  pgtbl_cfg->arm_lpae_s1_cfg.ttbr;
} else {
    cb->tcr[0]  = arm_smmu_lpae_vtcr(pgtbl_cfg);
    cb->ttbr[0] = pgtbl_cfg->arm_lpae_s2_cfg.vttbr;
}
```

### 6.4 Attach 设备

[`arm_smmu_attach_dev()`](../../linux/drivers/iommu/arm/arm-smmu/arm-smmu.c:1165)
把设备的 S2CR 指向新 domain 的 context bank：

```text
device master cfg
  -> arm_smmu_master_install_s2crs(
         type = S2CR_TYPE_TRANS,
         cbndx = domain.cfg.cbndx)
  -> 写 S2CR
```

Linux attach 的关键代码
（[arm-smmu.c](../../linux/drivers/iommu/arm/arm-smmu/arm-smmu.c:1165)）：

```c
ret = arm_smmu_init_domain_context(smmu_domain, smmu, dev);
if (ret < 0)
    goto rpm_put;

if (smmu_domain->smmu != smmu) {
    ret = -EINVAL;
    goto rpm_put;
}

arm_smmu_master_install_s2crs(cfg, S2CR_TYPE_TRANS,
                              smmu_domain->cfg.cbndx, fwspec);
```

`arm_smmu_master_install_s2crs()` 本身只做 S2CR 更新：

```c
static void arm_smmu_master_install_s2crs(struct arm_smmu_master_cfg *cfg,
                                          enum arm_smmu_s2cr_type type,
                                          u8 cbndx,
                                          struct iommu_fwspec *fwspec)
{
    struct arm_smmu_device *smmu = cfg->smmu;
    struct arm_smmu_s2cr *s2cr = smmu->s2crs;
    int i, idx;

    for_each_cfg_sme(cfg, fwspec, i, idx) {
        if (type == s2cr[idx].type && cbndx == s2cr[idx].cbndx)
            continue;

        s2cr[idx].type = type;
        s2cr[idx].privcfg = S2CR_PRIVCFG_DEFAULT;
        s2cr[idx].cbndx = cbndx;
        arm_smmu_write_s2cr(smmu, idx);
    }
    return 0;
}
```

所以 SMMUv2 的两级软件对象可以这样对应：

| 硬件概念 | Linux 对象 |
| --- | --- |
| SMMU instance | `struct arm_smmu_device` |
| 设备 master | `struct arm_smmu_master_cfg` |
| StreamID/mask entry | `smmu->smrs[]` + `smmu->s2crs[]` |
| translation context | `struct arm_smmu_domain` |
| context bank | `smmu->cbs[cbndx]` |
| pages | `smmu_domain->pgtbl_ops` |

### 6.5 Map/unmap

[`arm_smmu_map_pages()`](../../linux/drivers/iommu/arm/arm-smmu/arm-smmu.c:1266)
只是转调：

```text
smmu_domain->pgtbl_ops->map_pages()
```

实际 IOVA -> PA PTE 写入由 `io-pgtable-arm.c` 完成。这个分工很重要：

SMMU driver 管理 SMMU context 和 stream mapping；
`io-pgtable` 管理页表格式和 PTE。

### 6.6 Fault

[`arm_smmu_context_fault()`](../../linux/drivers/iommu/arm/arm-smmu/arm-smmu.c:455)
读取 context fault 信息，调用 `report_iommu_fault()`，再清 FSR 并决定是否
resume。没有上层 handler 时打印详细日志。

关键代码：

```c
arm_smmu_read_context_fault_info(smmu, idx, &cfi);

if (!(cfi.fsr & ARM_SMMU_CB_FSR_FAULT))
    return IRQ_NONE;

ret = report_iommu_fault(&smmu_domain->domain, NULL, cfi.iova,
                         cfi.fsynr & ARM_SMMU_CB_FSYNR0_WNR ?
                         IOMMU_FAULT_WRITE : IOMMU_FAULT_READ);

arm_smmu_cb_write(smmu, idx, ARM_SMMU_CB_FSR, cfi.fsr);

if (cfi.fsr & ARM_SMMU_CB_FSR_SS) {
    arm_smmu_cb_write(smmu, idx, ARM_SMMU_CB_RESUME,
                      ret == -EAGAIN ?
                      0 : ARM_SMMU_RESUME_TERMINATE);
}
```

## 7. SMMUv2 的 Xen 源码对应

### 7.1 探测与注册全局 ops

Xen 入口：

- [`arm_smmu_dt_init()`](../../xen/xen/drivers/passthrough/arm/smmu.c:2981)
- [`arm_smmu_device_dt_probe()`](../../xen/xen/drivers/passthrough/arm/smmu.c:2442)
- [`arm_smmu_device_cfg_probe()`](../../xen/xen/drivers/passthrough/arm/smmu.c:2253)

Xen 的 probe 同样读取 SMMU ID、context bank、SMR/S2CR 和地址能力，但随后
使用 Xen 自己的 `struct iommu_ops`：

```text
iommu_set_ops(&arm_smmu_iommu_ops)
```

其中关键回调是：

```text
.add_device       = arm_smmu_dt_add_device_generic
.assign_device    = arm_smmu_assign_dev
.reassign_device  = arm_smmu_reassign_dev
.map_page         = arm_iommu_map_page
.unmap_page       = arm_iommu_unmap_page
.iotlb_flush      = arm_smmu_iotlb_flush
```

Xen 注册给通用 iommu framework 的实际回调表如下
（[smmu.c](../../xen/xen/drivers/passthrough/arm/smmu.c:2947)）：

```c
static const struct iommu_ops arm_smmu_iommu_ops = {
    .page_sizes      = PAGE_SIZE_4K,
    .init            = arm_smmu_iommu_domain_init,
    .hwdom_init      = arch_iommu_hwdom_init,
    .add_device      = arm_smmu_dt_add_device_generic,
    .remove_device   = arm_smmu_dt_remove_device_generic,
    .teardown        = arm_smmu_iommu_domain_teardown,
    .iotlb_flush     = arm_smmu_iotlb_flush,
    .assign_device   = arm_smmu_assign_dev,
    .reassign_device = arm_smmu_reassign_dev,
    .map_page        = arm_iommu_map_page,
    .unmap_page      = arm_iommu_unmap_page,
    .dt_xlate        = arm_smmu_dt_xlate_generic,
};
```

### 7.2 登记设备

Xen 的
[`iommu_add_dt_device()`](../../xen/xen/drivers/passthrough/device_tree.c:258)
解析 `iommus` 后，SMMUv2 `dt_xlate` 将 StreamID 和 mask 填入 fwspec。
随后 `arm_smmu_dt_add_device_generic()`：

1. 从 fwspec 找到 SMMU；
2. 创建 `arm_smmu_master`；
3. 校验 StreamID；
4. 把 SMR/S2CR index 标为未分配；
5. 非 PCI 设备标记为 protected。

这与 Linux 的 `arm_smmu_probe_device()` + SMR/S2CR 初始化对应，但 Xen
额外维护设备是否属于 Xen、Dom0 或 DomU，以及是否已经 assignment。

Xen 的 `add_device` 关键部分是建立 master，并标记平台设备受 SMMU 保护
（[smmu.c](../../xen/xen/drivers/passthrough/arm/smmu.c:805)）：

```c
static int arm_smmu_dt_add_device_generic(u8 devfn, struct device *dev)
{
    struct arm_smmu_device *smmu;
    struct iommu_fwspec *fwspec;
    int ret;

    ...
    fwspec = dev_iommu_fwspec_get(dev);
    smmu = find_smmu(fwspec->iommu_dev);

    ret = arm_smmu_dt_add_device_legacy(smmu, dev, fwspec);
    if ( ret )
        return ret;

    ...
    return ret;
}
```

`arm_smmu_dt_add_device_legacy()` 内部对非 PCI 设备执行：

```c
if ( !dev_is_pci(dev) )
{
    /* Xen: Let Xen know that the device is protected by an SMMU */
    dt_device_set_protected(dev_node);
}
```

### 7.3 Domain 和 P2M

Xen 的 [`arm_smmu_assign_dev()`](../../xen/xen/drivers/passthrough/arm/smmu.c:2751)
在 target `struct domain` 上：

1. 查找该 domain 在目标 SMMU 上的 context；
2. 第一次使用时分配 IOMMU context；
3. attach 设备；
4. S2CR 指向 context bank；
5. context bank 的 TTBR0 使用
   `domain->arch.p2m.root`。

Xen `arm_smmu_assign_dev()` 的核心结构：

```c
static int arm_smmu_assign_dev(struct domain *d, u8 devfn,
                               struct device *dev, u32 flag)
{
    int ret = 0;
    struct iommu_domain *domain;
    struct arm_smmu_xen_domain *xen_domain = dom_iommu(d)->arch.priv;
    ...

    domain = arm_smmu_get_domain(d, dev);
    if (!domain) {
        domain = xzalloc(struct iommu_domain);
        ...
        ret = arm_smmu_domain_init(domain);
        ...
        domain->priv->cfg.domain = d;
        list_add(&domain->list, &xen_domain->contexts);
    }

    ret = arm_smmu_attach_dev(domain, dev);
    if (!ret) {
        atomic_inc(&domain->ref);
        dev_iommu_domain(dev) = domain;
    }
    ...
}
```

最关键区别在
[`arm_smmu_init_context_bank()`](../../xen/xen/drivers/passthrough/arm/smmu.c:1157)：

```c
stage1 = cfg->cbar != CBAR_TYPE_S2_TRANS;

if (stage1) {
    reg = cfg->cbar;
} else {
    reg = cfg->cbar | ARM_SMMU_CB_VMID(cfg) << CBAR_VMID_SHIFT;
}
...

/* Xen: The page table is shared with the P2M code */
p2maddr = page_to_maddr(smmu_domain->cfg.domain->arch.p2m.root);
reg64 = p2maddr;

if (stage1)
    reg64 |= ((uint64_t)ARM_SMMU_CB_ASID(cfg) << TTBRn_HI_ASID_SHIFT) << 32;

writeq_relaxed_non_atomic(reg64, cb_base + ARM_SMMU_CB_TTBR0);
```

S2CR 最终被指向这个 context bank：

```c
static int arm_smmu_domain_add_master(struct arm_smmu_domain *smmu_domain,
                                      struct arm_smmu_master_cfg *cfg)
{
    struct arm_smmu_device *smmu = smmu_domain->smmu;
    struct arm_smmu_s2cr *s2cr = smmu->s2crs;
    struct iommu_fwspec *fwspec = arm_smmu_get_fwspec(cfg);
    ...
    enum arm_smmu_s2cr_type type = S2CR_TYPE_TRANS;
    u8 cbndx = smmu_domain->cfg.cbndx;

    for_each_cfg_sme(cfg, i, idx, fwspec->num_ids) {
        s2cr[idx].type = type;
        s2cr[idx].privcfg = S2CR_PRIVCFG_DEFAULT;
        s2cr[idx].cbndx = cbndx;
        arm_smmu_write_s2cr(smmu, idx);
    }
}
```

Linux 的 TTBR0 来自 `io-pgtable` 新建的页表；Xen 的 TTBR0 来自 domain
P2M。SMMUv2 因此直接共享 Xen 的 CPU stage-2 页表。

### 7.4 Map/unmap

Xen `arm_smmu_iommu_ops.map_page` 指向
[`arm_iommu_map_page()`](../../xen/xen/drivers/passthrough/arm/iommu_helpers.c:33)。
它不是新建一张独立 IOMMU 页表，而是修改 Xen P2M：

```c
int __must_check arm_iommu_map_page(struct domain *d, dfn_t dfn, mfn_t mfn,
                                    unsigned int flags,
                                    unsigned int *flush_flags)
{
    p2m_type_t t;

    if ( !(flags & (IOMMUF_readable | IOMMUF_writable)) )
        return -EINVAL;

    t = (flags & IOMMUF_writable) ? p2m_iommu_map_rw : p2m_iommu_map_ro;

    return guest_physmap_add_entry(d, _gfn(dfn_x(dfn)), mfn,
                                   IOMMUF_order(flags), t);
}

int __must_check arm_iommu_unmap_page(struct domain *d, dfn_t dfn,
                                      unsigned int order,
                                      unsigned int *flush_flags)
{
    return guest_physmap_remove_page(d, _gfn(dfn_x(dfn)),
                                     INVALID_MFN, order);
}
```

因此：

```text
Linux map_page:
  SMMU domain page table PTE

Xen map_page:
  domain P2M entry
```

### 7.5 Fault 和 IOTLB

Xen SMMUv2 有 context/global fault handler，但当前没有 Linux 那样的完整
IOMMU fault 上报和用户态 fault queue 机制。

页表更新后的 SMMU TLB 失效由 Xen P2M 路径触发
[`iommu_iotlb_flush()`](../../xen/xen/arch/arm/mmu/p2m.c:1139)，
最终调用 [`arm_smmu_iotlb_flush()`](../../xen/xen/drivers/passthrough/arm/smmu.c:2691)
发出 SMMUv2 TLB invalidate。

Xen 版本没有把 fault 交给 Linux 的 `report_iommu_fault()`，而是直接记录：

```c
/* Xen doesn't handle IOMMU fault */
#define report_iommu_fault(...) 1

void arm_smmu_read_context_fault_info(struct arm_smmu_device *smmu, int idx,
                                      struct arm_smmu_context_fault_info *cfi)
{
    cfi->iova    = arm_smmu_cb_readq(smmu, idx, ARM_SMMU_CB_FAR);
    cfi->fsr     = arm_smmu_cb_read(smmu, idx, ARM_SMMU_CB_FSR);
    cfi->fsynr   = arm_smmu_cb_read(smmu, idx, ARM_SMMU_CB_FSYNR0);
    cfi->cbfrsynra = arm_smmu_gr1_read(smmu, ARM_SMMU_GR1_CBFRSYNRA(idx));
}
```

这再次说明 Xen 和 Linux 的 SMMU driver 共享硬件知识，但上层 fault
语义不同。

## 8. SMMUv3 详细原理

### 8.1 SMMUv3 的变化

SMMUv3 与 SMMUv2 的核心区别：

- Stream mapping 从大量 MMIO 寄存器迁移到内存中的 Stream Table；
- 每个 StreamID 对应一个 STE；
- STE 可以指向 stage-1 Context Descriptor table 或 stage-2 配置；
- 控制命令通过内存 Command Queue 提交；
- fault、PRI 通过 Event Queue 和 PRI Queue 上报；
- 原生支持 ATS、PRI，并可支持 PASID/substream。

### 8.2 Stream Table（STRTAB）

SMMUv3 以 StreamID 查找 STE：

```text
StreamID
  -> linear Stream Table
     或 two-level Stream Table
  -> STE
```

Stream Table 有线性模式和两级模式。驱动根据 SID 空间大小和硬件能力选择。

### 8.3 STE 的翻译模式

STE 的第 0 个 qword 中的 `CONFIG` 字段决定模式：

| 值 | 模式 | 含义 |
| --- | --- | --- |
| `0b000` | ABORT | 拒绝该 StreamID 的访问 |
| `0b100` | BYPASS | 不做翻译，地址直通 |
| `0b101` | S1_TRANS | stage-1 翻译 |
| `0b110` | S2_TRANS | stage-2 翻译 |
| `0b111` | NESTED | stage-1 + stage-2 |

Linux 定义可见
[SMMUv3 STE config](../../linux/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.h:249)。

对应代码就是：

```c
#define STRTAB_STE_0_CFG         GENMASK_ULL(3, 1)
#define STRTAB_STE_0_CFG_ABORT   0
#define STRTAB_STE_0_CFG_BYPASS  4
#define STRTAB_STE_0_CFG_S1_TRANS 5
#define STRTAB_STE_0_CFG_S2_TRANS 6
#define STRTAB_STE_0_CFG_NESTED  7
```

### 8.4 Stage-2 STE

stage-2 STE 主要包含：

```text
CONFIG = S2_TRANS
EATS   = ATS 是否启用
S2VMID = stage-2 地址空间 ID
VTCR   = stage-2 地址宽度/页大小/cache/shareability
S2TTB  = stage-2 根页表
S2S    = stall 支持
```

翻译过程：

```text
StreamID
  -> Stream Table
  -> STE.CONFIG = S2_TRANS
  -> STE.S2VMID + STE.VTCR + STE.S2TTB
  -> stage-2 page walk
  -> PA
```

### 8.5 Stage-1 STE 和 Context Descriptor

stage-1 STE 指向 Context Descriptor table：

```text
STE
  -> S1CTXPTR
  -> Context Descriptor
  -> stage-1 TTBR/TCR/ASID/MAIR
```

Context Descriptor 类似 SMMUv2 的 context bank 配置，但它位于内存中。
如果设备支持 PASID，一个 STE 可以通过多个 substream/Context Descriptor
支持多个地址空间。

### 8.6 Command Queue

SMMUv3 用 CMDQ 完成维护操作：

- `CFGI_STE`：失效一个 STE 配置 cache；
- `CFGI_CD`：失效一个 Context Descriptor cache；
- `CFGI_ALL`：失效全部配置 cache；
- `TLBI_*`：失效 TLB；
- `ATC_INV`：失效 PCIe ATS ATC；
- `PRI_RESP`：响应 PRI page request；
- `SYNC`：确认此前命令完成。

软件不能只改内存中的 STE/CD 就认为硬件立即看到新配置，需要 CMDQ 的 CFGI
和 SYNC。

### 8.7 Event Queue

EVTQ 报告：

- translation fault；
- configuration fault；
- command queue error；
- stream table fault；
- page request 相关的其他事件。

事件记录通常包含 StreamID、SSID/PASID、地址和权限信息。

### 8.8 PRI Queue

PCIe PRI 允许设备在页表未命中时请求软件填页，而不是立即失败。

```text
Device translation miss
  -> PRI request
  -> SMMU PRIQ
  -> software fault/page handler
  -> page mapped
  -> PRI response
  -> device retries
```

Linux 可以把 PRI 映射到 `io-pgtable`/IOPF/SVA 路径；Xen 当前主路径没有
完整的用户态 fault/page-request 管理。

### 8.9 ATS

PCIe ATS 允许 endpoint 向 SMMU 请求地址翻译，并把结果缓存到 ATC：

```text
Device ATC
  -> ATS translation request
  -> SMMU translation
  -> cached IOVA -> PA
```

软件修改 SMMU 页表后，不仅需要失效 SMMU TLB，还需要失效设备 ATC。
ATS 顺序错误会导致设备继续使用旧翻译。

## 9. SMMUv3 的 Linux 源码对应

### 9.1 Probe 和结构初始化

Linux 入口：

- [`arm_smmu_device_probe()`](../../linux/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c:5782)
- [`arm_smmu_device_hw_probe()`](../../linux/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c:5316)
- [`arm_smmu_init_structures()`](../../linux/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c:4858)
- [`arm_smmu_device_reset()`](../../linux/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c:5102)

初始化顺序概括：

```text
DT/ACPI probe
  -> read ID registers
  -> discover SID bits / feature bits / queue limits
  -> allocate command queue
  -> allocate event queue
  -> allocate PRI queue if present
  -> allocate Stream Table
  -> set base registers
  -> enable queues and SMMU
  -> invalidate configuration and TLB
```

`arm_smmu_init_structures()` 的核心：

```c
int arm_smmu_init_structures(struct arm_smmu_device *smmu)
{
    mutex_init(&smmu->streams_mutex);
    smmu->streams = RB_ROOT;

    ret = arm_smmu_init_queues(smmu);
    if (ret)
        return ret;

    ret = arm_smmu_init_strtab(smmu);
    if (ret)
        return ret;

    ...
    return 0;
}
```

`arm_smmu_device_reset()` 把内存结构接到硬件：

```c
arm_smmu_write_strtab(smmu);

writeq_relaxed(smmu->cmdq.q.q_base,
               smmu->base + ARM_SMMU_CMDQ_BASE);
enables = CR0_CMDQEN;
arm_smmu_write_reg_sync(smmu, enables, ARM_SMMU_CR0,
                        ARM_SMMU_CR0ACK);

arm_smmu_cmdq_issue_cmd_with_sync(
    smmu, arm_smmu_make_cmd_cfgi_all());
arm_smmu_cmdq_issue_cmd_with_sync(
    smmu, arm_smmu_make_cmd_op(CMDQ_OP_TLBI_NSNH_ALL));

enables |= CR0_EVTQEN;
arm_smmu_write_reg_sync(smmu, enables, ARM_SMMU_CR0,
                        ARM_SMMU_CR0ACK);
...
enables |= CR0_SMMUEN;
arm_smmu_write_reg_sync(smmu, enables, ARM_SMMU_CR0,
                        ARM_SMMU_CR0ACK);
```

### 9.2 登记设备

[`arm_smmu_probe_device()`](../../linux/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c:4430)
建立：

```text
struct arm_smmu_master
  -> device
  -> SMMU
  -> streams[]
  -> cd_table
```

[`arm_smmu_insert_master()`](../../linux/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c:4285)
把设备的每个 StreamID 加入 SMMU 的 SID RB-tree：

```text
StreamID
  -> 校验范围
  -> 初始化需要的二级 Stream Table
  -> 检查 SID alias
  -> 插入 master
```

关键源码：

```c
static int arm_smmu_insert_master(struct arm_smmu_device *smmu,
                                  struct arm_smmu_master *master)
{
    struct iommu_fwspec *fwspec = dev_iommu_fwspec_get(master->dev);
    ...

    master->streams = kzalloc_objs(*master->streams, fwspec->num_ids);
    master->num_streams = fwspec->num_ids;

    for (i = 0; i < fwspec->num_ids; i++) {
        struct arm_smmu_stream *new_stream = &master->streams[i];

        new_stream->id = fwspec->ids[i];
        new_stream->master = master;
    }

    mutex_lock(&smmu->streams_mutex);
    for (i = 0; i < fwspec->num_ids; i++) {
        struct arm_smmu_stream *new_stream = &master->streams[i];
        u32 sid = new_stream->id;

        ret = arm_smmu_init_sid_strtab(smmu, sid);
        if (ret)
            break;

        existing = rb_find_add(&new_stream->node, &smmu->streams,
                               arm_smmu_streams_cmp_node);
        ...
    }
    mutex_unlock(&smmu->streams_mutex);
    ...
}
```

### 9.3 Domain finalise

Linux 在
[`arm_smmu_domain_alloc_paging_flags()`](../../linux/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c:4121)
中先决定 stage：普通 paging domain 优先 stage-1；硬件没有 stage-1 时回退
stage-2；nested parent 明确使用 stage-2。

[`arm_smmu_domain_finalise()`](../../linux/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c:3061)
根据 domain stage：

- stage-1：分配 ASID，建立 `ARM_64_LPAE_S1` 页表；
- stage-2：分配 VMID，建立 `ARM_64_LPAE_S2` 页表。

`alloc_io_pgtable_ops()` 返回页表对象，其配置中已经包含：

- stage-1 的 TCR/TTBR/MAIR；
- stage-2 的 VTCR/VTTBR。

Linux domain finalise 的关键代码：

```c
switch (smmu_domain->stage) {
case ARM_SMMU_DOMAIN_S1:
    pgtbl_cfg.ias = min_t(unsigned long, ias, VA_BITS);
    pgtbl_cfg.oas = smmu->oas;
    fmt = ARM_64_LPAE_S1;
    finalise_stage_fn = arm_smmu_domain_finalise_s1;
    break;

case ARM_SMMU_DOMAIN_S2:
    pgtbl_cfg.ias = smmu->oas;
    pgtbl_cfg.oas = smmu->oas;
    fmt = ARM_64_LPAE_S2;
    finalise_stage_fn = arm_smmu_domain_finalise_s2;
    break;
}

pgtbl_ops = alloc_io_pgtable_ops(fmt, &pgtbl_cfg, smmu_domain);
ret = finalise_stage_fn(smmu, smmu_domain);
smmu_domain->pgtbl_ops = pgtbl_ops;
smmu_domain->smmu = smmu;
```

`arm_smmu_domain_finalise_s1/s2()` 分别分配 ASID 或 VMID；页表本身由
`alloc_io_pgtable_ops()` 建立。

### 9.4 Attach 和 STE 写入

[`arm_smmu_attach_dev()`](../../linux/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c:3805)
根据 domain stage 构造目标配置：

- stage-1：`arm_smmu_make_s1_cd()` + `arm_smmu_make_cdtable_ste()`；
- stage-2：`arm_smmu_make_s2_domain_ste()`。

[`arm_smmu_make_s2_domain_ste()`](../../linux/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c:1963)
把 `VTCR/VTTBR/VMID` 组装成 STE。

[`arm_smmu_install_ste_for_dev()`](../../linux/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c:3144)
对设备的每个 StreamID 找到对应 STE，并调用 STE 更新逻辑。

Linux attach 的关键 switch：

```c
switch (smmu_domain->stage) {
case ARM_SMMU_DOMAIN_S1:
    arm_smmu_make_s1_cd(&target_cd, master, smmu_domain);
    arm_smmu_write_cd_entry(master, IOMMU_NO_PASID, cdptr, &target_cd);
    arm_smmu_make_cdtable_ste(&target, master, state.ats_enabled,
                              STRTAB_STE_1_S1DSS_SSID0);
    arm_smmu_install_ste_for_dev(master, &target);
    break;

case ARM_SMMU_DOMAIN_S2:
    arm_smmu_make_s2_domain_ste(&target, master, smmu_domain,
                                state.ats_enabled);
    arm_smmu_install_ste_for_dev(master, &target);
    arm_smmu_clear_cd(master, IOMMU_NO_PASID);
    break;
}
```

`arm_smmu_make_s2_domain_ste()` 中直接构造 STE：

```c
target->data[0] = cpu_to_le64(
    STRTAB_STE_0_V |
    FIELD_PREP(STRTAB_STE_0_CFG, STRTAB_STE_0_CFG_S2_TRANS));

target->data[1] = cpu_to_le64(
    FIELD_PREP(STRTAB_STE_1_EATS,
               ats_enabled ? STRTAB_STE_1_EATS_TRANS : 0) |
    ...);

vtcr_val = FIELD_PREP(STRTAB_STE_2_VTCR_S2T0SZ, vtcr->tsz) |
           FIELD_PREP(STRTAB_STE_2_VTCR_S2SL0, vtcr->sl) |
           ...
           FIELD_PREP(STRTAB_STE_2_VTCR_S2PS, vtcr->ps);

target->data[2] = cpu_to_le64(
    FIELD_PREP(STRTAB_STE_2_S2VMID, s2_cfg->vmid) |
    FIELD_PREP(STRTAB_STE_2_VTCR, vtcr_val) |
    STRTAB_STE_2_S2AA64 |
    STRTAB_STE_2_S2PTW |
    STRTAB_STE_2_S2R);

target->data[3] = cpu_to_le64(
    pgtbl_cfg->arm_lpae_s2_cfg.vttbr & STRTAB_STE_3_S2TTB_MASK);
```

### 9.5 STE 更新和内存顺序

SMMUv3 的 STE/CD 是多 qword 结构，而 CPU 只保证单个对齐 64-bit store
原子可见。驱动不能用普通 memcpy 随意修改 live STE。

[`arm_smmu_write_entry()`](../../linux/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c:1459)
通过分析哪些 qword 的“有效 bit”变化，选择：

- hitless update；
- 先清 V，再写其它 qword，最后写 V 的 breaking update。

每次 live 配置变化后，通过 CMDQ 发 `CFGI_STE`/`CFGI_CD` 和 `SYNC`。

核心代码：

```c
dma_wmb();

used_qword_diff =
    arm_smmu_entry_qword_diff(writer, entry, target, unused_update);

if (hweight8(used_qword_diff) == 1) {
    /* Hitless：先写不生效的字段，再写一个关键 qword */
    unsigned int critical_qword_index = ffs(used_qword_diff) - 1;

    unused_update[critical_qword_index] =
        entry[critical_qword_index];
    entry_set(writer, entry, unused_update, 0, NUM_ENTRY_QWORDS);
    entry_set(writer, entry, target, critical_qword_index, 1);
    entry_set(writer, entry, target, 0, NUM_ENTRY_QWORDS);
} else if (used_qword_diff) {
    /* 必须中断：先清 V，再写其它 qword，最后恢复 V */
    unused_update[0] = 0;
    entry_set(writer, entry, unused_update, 0, 1);
    entry_set(writer, entry, target, 1, NUM_ENTRY_QWORDS - 1);
    entry_set(writer, entry, target, 0, 1);
} else {
    entry_set(writer, entry, target, 0, NUM_ENTRY_QWORDS);
}
```

STE writer 的 `sync()` 回调执行：

```c
arm_smmu_cmdq_issue_cmd_with_sync(
    writer->master->smmu,
    arm_smmu_make_cmd_cfgi_ste(ste_writer->sid, true));
```

### 9.6 Map/unmap 和 TLB

[`arm_smmu_map_pages()`](../../linux/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c:4184)
转调 `smmu_domain->pgtbl_ops->map_pages()`。

页表实际由 `io-pgtable-arm.c` 修改。TLB sync 使用
[`arm_smmu_iotlb_sync()`](../../linux/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c:4217)
和 `CMDQ_OP_TLBI_*`。

### 9.7 Fault、PRI、ATS、PASID

- EVTQ：`arm_smmu_evtq_thread()`；
- PRIQ：`arm_smmu_priq_thread()`；
- PPR：`arm_smmu_handle_ppr()`；
- ATS enable：`arm_smmu_enable_ats()`；
- PASID/SSID：`arm_smmu_enable_pasid()` 和 CD table；
- SVA/IOPF：`arm-smmu-v3-sva.c` 及 `io-pgfault.c`。

EVTQ 事件首先被解码成软件字段：

```c
event->id    = FIELD_GET(EVTQ_0_ID, raw[0]);
event->sid   = FIELD_GET(EVTQ_0_SID, raw[0]);
event->ssv   = FIELD_GET(EVTQ_0_SSV, raw[0]);
event->ssid  = event->ssv ?
               FIELD_GET(EVTQ_0_SSID, raw[0]) : IOMMU_NO_PASID;
event->s2    = FIELD_GET(EVTQ_1_S2, raw[1]);
event->read  = FIELD_GET(EVTQ_1_RnW, raw[1]);
event->stall = FIELD_GET(EVTQ_1_STALL, raw[1]);
event->iova  = FIELD_GET(EVTQ_2_ADDR, raw[2]);
```

stall/page-request 会转换成 Linux fault event：

```c
if (event->stall) {
    flt->type = IOMMU_FAULT_PAGE_REQ;
    flt->prm = (struct iommu_fault_page_request){
        .flags = IOMMU_FAULT_PAGE_REQUEST_LAST_PAGE,
        .grpid = event->stag,
        .perm  = perm,
        .addr  = event->iova,
    };
}

master = arm_smmu_find_master(smmu, event->sid);
if (event->stall)
    ret = iommu_report_device_fault(master->dev, &fault_evt);
```

Linux 启用 PASID 的核心代码：

```c
static int arm_smmu_enable_pasid(struct arm_smmu_master *master)
{
    struct pci_dev *pdev;
    int features;
    int num_pasids;
    int ret;

    if (!dev_is_pci(master->dev))
        return -ENODEV;

    pdev = to_pci_dev(master->dev);
    features = pci_pasid_features(pdev);
    if (features < 0)
        return features;

    num_pasids = pci_max_pasids(pdev);
    if (num_pasids <= 0)
        return num_pasids;

    ret = pci_enable_pasid(pdev, features);
    if (ret)
        return ret;

    master->ssid_bits = min_t(u8, ilog2(num_pasids),
                              master->smmu->ssid_bits);
    return 0;
}
```

stage-1 Context Descriptor 把页表根和 ASID 写入 CD：

```c
target->data[0] = cpu_to_le64(
    FIELD_PREP(CTXDESC_CD_0_TCR_T0SZ, tcr->tsz) |
    FIELD_PREP(CTXDESC_CD_0_TCR_TG0, tcr->tg) |
    CTXDESC_CD_0_V |
    CTXDESC_CD_0_AA64 |
    FIELD_PREP(CTXDESC_CD_0_ASID, cd->asid));

target->data[1] = cpu_to_le64(
    pgtbl_cfg->arm_lpae_s1_cfg.ttbr & CTXDESC_CD_1_TTB0_MASK);
```

ATS 的软件入口：

```c
static void arm_smmu_enable_ats(struct arm_smmu_master *master)
{
    size_t stu = __ffs(master->smmu->pgsize_bitmap);
    struct pci_dev *pdev = to_pci_dev(master->dev);

    arm_smmu_atc_inv_master(master, IOMMU_NO_PASID);
    pci_enable_ats(pdev, stu);
}
```

因此 Linux SMMUv3 不只是“DMA 翻译驱动”，还是一个支持 fault、
page-request、PASID 和 SVA 的 IOMMU 子系统。

## 10. SMMUv3 的 Xen 源码对应

### 10.1 Probe 与全局 ops

Xen 入口：

- [`arm_smmu_dt_init()`](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:2872)
- [`arm_smmu_device_probe()`](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:2477)
- [`arm_smmu_init_structures()`](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:1795)
- [`arm_smmu_device_reset()`](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:2060)

Xen 的 SMMUv3 文件头部已经明确列出与 Linux 的主要差异：

- 当前是 Tech Preview；
- 只支持 stage-2；
- 共享 Xen P2M，不新建 IOMMU 页表；
- 使用 tasklet/lock 的较早实现；
- ATS/MSI 代码不是完整生产支持。

Xen `arm_smmu_init_structures()` 与 Linux 的结构一致：

```c
static int __init arm_smmu_init_structures(struct arm_smmu_device *smmu)
{
    ret = arm_smmu_init_queues(smmu);
    if (ret)
        return ret;

    ret = arm_smmu_init_strtab(smmu);
    if (ret)
        return ret;

    return 0;
}
```

`arm_smmu_device_reset()` 同样执行：

```text
写 STRTAB_BASE
  -> 启用 CMDQ
  -> CFGI_ALL
  -> TLBI
  -> 初始化 EVTQ/PRIQ
  -> 启用 SMMUEN
```

### 10.2 设备 StreamID

[`arm_smmu_add_device()`](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:1496)
：

1. 从 `iommu_fwspec` 取得 SID 列表；
2. 创建 `arm_smmu_master`；
3. 检查 StreamID 在 Stream Table 范围内；
4. 必要时初始化二级 Stream Table；
5. 标记非 PCI 设备 protected；
6. 记录 master。

关键源码：

```c
static int arm_smmu_add_device(u8 devfn, struct device *dev)
{
    int i, ret;
    struct arm_smmu_device *smmu;
    struct arm_smmu_master *master;
    struct iommu_fwspec *fwspec;

    fwspec = dev_iommu_fwspec_get(dev);
    if (!fwspec)
        return -ENODEV;

    smmu = arm_smmu_get_by_dev(fwspec->iommu_dev);
    if (!smmu)
        return -ENODEV;

    master = xzalloc(struct arm_smmu_master);
    master->dev = dev;
    master->smmu = smmu;
    master->sids = fwspec->ids;
    master->num_sids = fwspec->num_ids;
    dev_iommu_priv_set(dev, master);

    for (i = 0; i < master->num_sids; i++) {
        u32 sid = master->sids[i];

        if (!arm_smmu_sid_in_range(smmu, sid))
            return -ERANGE;
    }

    if ( !dev_is_pci(dev) )
        dt_device_set_protected(dev_to_dt(dev));

    return 0;
}
```

### 10.3 Domain 和 P2M

[`arm_smmu_domain_finalise_s2()`](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:1163)
：

```c
arm_lpae_s2_cfg.vttbr =
    page_to_maddr(smmu_domain->d->arch.p2m.root);
```

并计算：

```text
S2VMID
VTCR.T0SZ
VTCR.SL0
VTCR.IRGN/ORGN/SH/TG/PS
```

[`arm_smmu_write_strtab_ent()`](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:666)
再把配置写入 STE：

```text
CONFIG = S2_TRANS
S2VMID
VTCR
S2TTB = P2M root
```

Xen `arm_smmu_domain_finalise_s2()` 的关键代码：

```c
static int arm_smmu_domain_finalise_s2(struct arm_smmu_domain *smmu_domain,
                                       struct arm_smmu_master *master)
{
    int vmid;
    struct arm_lpae_s2_cfg arm_lpae_s2_cfg;
    struct arm_smmu_s2_cfg *cfg = &smmu_domain->s2_cfg;
    struct arm_smmu_device *smmu = smmu_domain->smmu;
    typeof(&arm_lpae_s2_cfg.vtcr) vtcr = &arm_lpae_s2_cfg.vtcr;

    vtcr->sh   = ARM_SMMU_VTCR_SH_IS;
    vtcr->irgn = ARM_SMMU_VTCR_RGN_WBWA;
    vtcr->orgn = ARM_SMMU_VTCR_RGN_WBWA;
    vtcr->tg   = ARM_SMMU_VTCR_TG0_4K;

    switch (smmu->oas) {
    case 32: vtcr->ps = ARM_SMMU_VTCR_PS_32_BIT; break;
    case 36: vtcr->ps = ARM_SMMU_VTCR_PS_36_BIT; break;
    case 40: vtcr->ps = ARM_SMMU_VTCR_PS_40_BIT; break;
    case 42: vtcr->ps = ARM_SMMU_VTCR_PS_42_BIT; break;
    case 44: vtcr->ps = ARM_SMMU_VTCR_PS_44_BIT; break;
    case 48: vtcr->ps = ARM_SMMU_VTCR_PS_48_BIT; break;
    case 52: vtcr->ps = ARM_SMMU_VTCR_PS_52_BIT; break;
    default: return -EINVAL;
    }

    vtcr->tsz  = 64 - p2m_ipa_bits;
    vtcr->sl   = 2 - P2M_ROOT_LEVEL;

    arm_lpae_s2_cfg.vttbr =
        page_to_maddr(smmu_domain->d->arch.p2m.root);

    vmid = arm_smmu_bitmap_alloc(smmu->vmid_map, smmu->vmid_bits);
    if (vmid < 0)
        return vmid;

    cfg->vmid  = (u16)vmid;
    cfg->vttbr = arm_lpae_s2_cfg.vttbr;
    cfg->vtcr  = FIELD_PREP(STRTAB_STE_2_VTCR_S2T0SZ, vtcr->tsz) |
                 FIELD_PREP(STRTAB_STE_2_VTCR_S2SL0, vtcr->sl) |
                 FIELD_PREP(STRTAB_STE_2_VTCR_S2IR0, vtcr->irgn) |
                 FIELD_PREP(STRTAB_STE_2_VTCR_S2OR0, vtcr->orgn) |
                 FIELD_PREP(STRTAB_STE_2_VTCR_S2SH0, vtcr->sh) |
                 FIELD_PREP(STRTAB_STE_2_VTCR_S2TG, vtcr->tg) |
                 FIELD_PREP(STRTAB_STE_2_VTCR_S2PS, vtcr->ps);
    return 0;
}
```

Xen 写 stage-2 STE 的实际代码
（[smmu-v3.c](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:666)）：

```c
u64 val = STRTAB_STE_0_V;

if (s2_cfg) {
    u64 strtab =
        FIELD_PREP(STRTAB_STE_2_S2VMID, s2_cfg->vmid) |
        FIELD_PREP(STRTAB_STE_2_VTCR, s2_cfg->vtcr) |
        STRTAB_STE_2_S2PTW | STRTAB_STE_2_S2AA64 |
        STRTAB_STE_2_S2R;

    dst[2] = cpu_to_le64(strtab);
    dst[3] = cpu_to_le64(s2_cfg->vttbr & STRTAB_STE_3_S2TTB_MASK);

    val |= FIELD_PREP(STRTAB_STE_0_CFG, STRTAB_STE_0_CFG_S2_TRANS);
}

if (master->ats_enabled)
    dst[1] |= cpu_to_le64(FIELD_PREP(STRTAB_STE_1_EATS,
                                     STRTAB_STE_1_EATS_TRANS));

write_atomic(&dst[0], cpu_to_le64(val));
```

### 10.4 Attach

[`arm_smmu_assign_dev()`](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:2685)
按 SMMU 为 Xen domain 查找或创建 `arm_smmu_domain`，然后：

```text
arm_smmu_attach_dev()
  -> domain finalise if first attach
  -> install STE for all SIDs
  -> add master to domain devices list
```

`assign_device` 中建立/复用 context 的代码：

```c
static int arm_smmu_assign_dev(struct domain *d, u8 devfn,
                               struct device *dev, u32 flag)
{
    int ret = 0;
    struct iommu_domain *io_domain;
    struct arm_smmu_domain *smmu_domain;
    struct arm_smmu_xen_domain *xen_domain = dom_iommu(d)->arch.priv;

    ...
    spin_lock(&xen_domain->lock);

    io_domain = arm_smmu_get_domain(d, dev);
    if (!io_domain) {
        io_domain = arm_smmu_domain_alloc();
        smmu_domain = to_smmu_domain(io_domain);
        smmu_domain->d = d;
        list_add(&io_domain->list, &xen_domain->contexts);
    }

    ret = arm_smmu_attach_dev(io_domain, dev);
    if (!ret)
        atomic_inc(&io_domain->ref);

    spin_unlock(&xen_domain->lock);
    return ret;
}
```

### 10.5 Map/unmap

Xen 的 `arm_smmu_iommu_ops` 仍指向：

```text
.map_page   = arm_iommu_map_page
.unmap_page = arm_iommu_unmap_page
```

也就是修改 P2M，而不是 `io-pgtable-arm.c` 的独立 IOMMU 页表。

### 10.6 Invalidate

Xen [`arm_smmu_iotlb_flush_all()`](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:2607)
遍历该 Xen domain 在该 SMMU 上的 context，并调用
`arm_smmu_tlb_inv_context()`：

```text
CMDQ_OP_TLBI_S12_VMALL
CMDQ_OP_SYNC
```

对应代码：

```c
static int __must_check arm_smmu_iotlb_flush_all(struct domain *d)
{
    struct arm_smmu_xen_domain *xen_domain = dom_iommu(d)->arch.priv;
    struct iommu_domain *io_domain;

    spin_lock(&xen_domain->lock);

    list_for_each_entry(io_domain, &xen_domain->contexts, list) {
        if (unlikely(!ACCESS_ONCE(to_smmu_domain(io_domain)->smmu)))
            continue;

        arm_smmu_tlb_inv_context(to_smmu_domain(io_domain));
    }

    spin_unlock(&xen_domain->lock);
    return 0;
}
```

`arm_smmu_tlb_inv_context()` 则提交 TLBI 和 SYNC：

```c
cmd.opcode = CMDQ_OP_TLBI_S12_VMALL;
cmd.tlbi.vmid = smmu_domain->s2_cfg.vmid;

arm_smmu_cmdq_issue_cmd(smmu, &cmd);
arm_smmu_cmdq_issue_sync(smmu);
```

### 10.7 Fault、PRI 和 ATS 边界

Xen SMMUv3 有 EVTQ/PRIQ tasklet，可以记录 event 和 PRI request，也会对
unexpected PRI 返回 deny。

Xen EVTQ tasklet 当前主要记录原始 event：

```c
static void arm_smmu_evtq_tasklet(void *dev)
{
    int i;
    struct arm_smmu_device *smmu = dev;
    struct arm_smmu_queue *q = &smmu->evtq.q;
    u64 evt[EVTQ_ENT_DWORDS];

    do {
        while (!queue_remove_raw(q, evt)) {
            u8 id = FIELD_GET(EVTQ_0_ID, evt[0]);

            dev_info(smmu->dev, "event 0x%02x received:\n", id);
            for (i = 0; i < ARRAY_SIZE(evt); ++i)
                dev_info(smmu->dev, "\t0x%016llx\n",
                         (unsigned long long)evt[i]);
        }
        ...
    } while (!queue_empty(&q->llq));
}
```

但是当前文件头明确说明 ATS 功能没有 Xen 侧测试/编译支持，PRI/fault 也没有
Linux 那样完整的 IOPF 用户态处理。阅读时不要把 Linux 的 SVA 能力直接映射
到 Xen。

文件头原文：

```text
 *  7. PCI ATS functionality is not supported, as there is no support available
 *     in XEN to test the functionality. Code is not tested and compiled. Code
 *     is guarded by the flag CONFIG_PCI_ATS.
```

遇到 PRI request 时，Xen 当前只记录并拒绝：

```c
static void arm_smmu_handle_ppr(struct arm_smmu_device *smmu, u64 *evt)
{
    u32 sid, ssid;
    u16 grpid;
    bool ssv, last;

    sid = FIELD_GET(PRIQ_0_SID, evt[0]);
    ssv = FIELD_GET(PRIQ_0_SSID_V, evt[0]);
    ssid = ssv ? FIELD_GET(PRIQ_0_SSID, evt[0]) : 0;
    last = FIELD_GET(PRIQ_0_PRG_LAST, evt[0]);
    grpid = FIELD_GET(PRIQ_1_PRG_IDX, evt[1]);

    ...
    dev_info(smmu->dev, "unexpected PRI request received:\n");
    if (last) {
        struct arm_smmu_cmdq_ent cmd = {
            .opcode = CMDQ_OP_PRI_RESP,
            .substream_valid = ssv,
            .pri = {
                .ssid  = ssid,
                .sid   = sid,
                .grpid = grpid,
                .resp  = PRI_RESP_DENY,
            },
        };
        arm_smmu_cmdq_issue_cmd(smmu, &cmd);
    }
}
```

## 11. 页表格式与 map/unmap 的本质

### 11.1 SMMUv2 的 descriptor 格式

SMMUv2 可以支持：

- ARMv7 short descriptor；
- ARMv8 LPAE stage-1；
- ARMv8 LPAE stage-2。

Linux 的 `arm_smmu_init_domain_context()` 根据 SMMU 能力、CPU 架构和
domain stage 选择格式，再调用 `alloc_io_pgtable_ops()`。

Xen SMMUv2 主要使用 P2M 本身的 LPAE 表格式；它不单独选择另一套 stage-2
页表格式。

### 11.2 LPAE 页表

LPAE 使用多级页表，叶子 entry 可以是：

- page；
- block；
- table。

stage-1 和 stage-2 的权限字段不完全相同。Linux `io-pgtable-arm.c` 在
[`arm_lpae_prot_to_pte()`](../../linux/drivers/iommu/io-pgtable-arm.c:516)
中把通用 `IOMMU_READ/WRITE/CACHE/MMIO/NOEXEC/PRIV` 转成 stage-1 或
stage-2 PTE。

关键分支：

```c
if (data->iop.fmt == ARM_64_LPAE_S1 ||
    data->iop.fmt == ARM_32_LPAE_S1) {
    pte = ARM_LPAE_PTE_nG;
    if (!(prot & IOMMU_WRITE) && (prot & IOMMU_READ))
        pte |= ARM_LPAE_PTE_AP_RDONLY;
    ...
} else {
    pte = ARM_LPAE_PTE_HAP_FAULT;
    if (prot & IOMMU_READ)
        pte |= ARM_LPAE_PTE_HAP_READ;
    if (prot & IOMMU_WRITE)
        pte |= ARM_LPAE_PTE_HAP_WRITE;
}

if (data->iop.fmt == ARM_64_LPAE_S2 ||
    data->iop.fmt == ARM_32_LPAE_S2) {
    if (prot & IOMMU_MMIO)
        pte |= ARM_LPAE_PTE_MEMATTR_DEV;
    else if (prot & IOMMU_CACHE)
        pte |= ARM_LPAE_PTE_MEMATTR_OIWB;
    else
        pte |= ARM_LPAE_PTE_MEMATTR_NC;
} else {
    if (prot & IOMMU_MMIO)
        pte |= ARM_LPAE_MAIR_ATTR_IDX_DEV <<
               ARM_LPAE_PTE_ATTRINDX_SHIFT;
    else if (prot & IOMMU_CACHE)
        pte |= ARM_LPAE_MAIR_ATTR_IDX_CACHE <<
               ARM_LPAE_PTE_ATTRINDX_SHIFT;
}

if (prot & IOMMU_NOEXEC)
    pte |= ARM_LPAE_PTE_XN;

pte |= ARM_LPAE_PTE_AF;
```

### 11.3 Linux io-pgtable

主要函数：

- [`arm_lpae_map_pages()`](../../linux/drivers/iommu/io-pgtable-arm.c:583)
- [`__arm_lpae_map()`](../../linux/drivers/iommu/io-pgtable-arm.c:456)
- [`arm_lpae_unmap_pages()`](../../linux/drivers/iommu/io-pgtable-arm.c:718)
- [`arm_lpae_iova_to_phys()`](../../linux/drivers/iommu/io-pgtable-arm.c:768)

流程：

```text
iommu_map(domain, iova, paddr, prot)
  -> domain->ops->map_pages()
  -> arm_smmu_map_pages()
  -> pgtbl_ops->map_pages()
  -> arm_lpae_map_pages()
  -> 分配中间页表
  -> 写 PTE
  -> cache maintenance / TLB invalidation
```

`arm_smmu_map_pages()` 只负责把请求交给 domain 的页表对象：

```c
static int arm_smmu_map_pages(struct iommu_domain *domain,
                              unsigned long iova, phys_addr_t paddr,
                              size_t pgsize, size_t pgcount,
                              int prot, gfp_t gfp, size_t *mapped)
{
    struct io_pgtable_ops *ops =
        to_smmu_domain(domain)->pgtbl_ops;

    if (!ops)
        return -ENODEV;

    return ops->map_pages(ops, iova, paddr, pgsize, pgcount,
                          prot, gfp, mapped);
}
```

`io-pgtable-arm.c` 中真正的递归 map：

```c
static int __arm_lpae_map(..., unsigned long iova, phys_addr_t paddr,
                          size_t size, ..., int lvl, arm_lpae_iopte *ptep,
                          gfp_t gfp, size_t *mapped)
{
    size_t block_size = ARM_LPAE_BLOCK_SIZE(lvl, data);

    if (size == block_size) {
        ret = arm_lpae_init_pte(data, iova, paddr, prot,
                                lvl, num_entries, ptep);
        if (!ret)
            *mapped += num_entries * size;
        return ret;
    }

    pte = READ_ONCE(*ptep);
    if (!pte) {
        cptep = __arm_lpae_alloc_pages(tblsz, gfp, cfg, cookie);
        pte = arm_lpae_install_table(cptep, ptep, 0, data);
    } else {
        cptep = iopte_deref(pte, data);
    }

    return __arm_lpae_map(..., iova, paddr, size, ...,
                          lvl + 1, cptep, gfp, mapped);
}
```

它体现的就是 ARM LPAE 页表：当前 level 能覆盖整个请求时写 leaf
page/block entry，否则分配下一级 table 继续向下走。

### 11.4 Xen P2M

Xen 路径：

```text
guest_physmap_add_entry()
  -> p2m_set_entry()
  -> 写 P2M entry
  -> iommu_iotlb_flush()
```

P2M entry 已经携带：

- MFN；
- memory type；
- access permission；
- IOMMU MMIO 属性。

SMMU driver 无须再把这些信息复制到第二张 stage-2 页表。

Xen `p2m_set_entry()` 更新 entry 后，同一函数中立即触发 IOMMU TLB flush
（[mmu/p2m.c](../../xen/xen/arch/arm/mmu/p2m.c:1139)）：

```c
if ( is_iommu_enabled(p2m->domain) &&
     (lpae_is_valid(orig_pte) || lpae_is_valid(*entry)) )
{
    unsigned int flush_flags = 0;

    if ( lpae_is_valid(orig_pte) )
        flush_flags |= IOMMU_FLUSHF_modified;
    if ( lpae_is_valid(*entry) )
        flush_flags |= IOMMU_FLUSHF_added;

    rc = iommu_iotlb_flush(p2m->domain, _dfn(gfn_x(sgfn)),
                           1UL << page_order, flush_flags);
}
```

因此 Xen 的“IOMMU map/unmap”经常并不经过 `iommu_map()`，而是直接在 CPU
stage-2/P2M 更新路径中完成，再同步 SMMU TLB。

## 12. DMA API 和 SMMU 的关系

### 12.1 Linux host：DMA API 是上层入口

Linux 设备驱动通常调用：

```text
dma_map_single()
dma_map_page()
dma_map_sg()
```

有 IOMMU 时，DMA API 通过 IOMMU DMA ops 分配 IOVA 并建立翻译：

```text
Driver
  -> dma_map_sg()
  -> iommu_dma_map_sg()
  -> iommu_dma_alloc_iova()
  -> iommu_map_sg()
  -> iommu_map()
  -> SMMU map_pages()
  -> io-pgtable
  -> SMMU page walk
```

入口可见：

- [`iommu_dma_map_sg()`](../../linux/drivers/iommu/dma-iommu.c:1413)
- [`iommu_map()`](../../linux/drivers/iommu/iommu.c:2708)
- [`iommu_setup_dma_ops()`](../../linux/drivers/iommu/dma-iommu.c:2166)

DMA API 还负责：

- IOVA allocator；
- address mask/segment boundary；
- cache sync；
- SWIOTLB；
- reserved regions；
- MSI IOVA；
- mapping lifetime。

这些都不是 SMMU 硬件自动完成的。

Linux `iommu_dma_map_sg()` 的关键步骤
（[dma-iommu.c](../../linux/drivers/iommu/dma-iommu.c:1413)）：

```c
for_each_sg(sg, s, nents, i) {
    size_t s_iova_off = iova_offset(iovad, s->offset);
    size_t s_length = s->length;

    sg_dma_address(s) = s_iova_off;
    sg_dma_len(s) = s_length;
    s->offset -= s_iova_off;
    s_length = iova_align(iovad, s_length + s_iova_off);
    s->length = s_length;
    iova_len += s_length;
}

iova = iommu_dma_alloc_iova(domain, iova_len,
                            dma_get_mask(dev), dev);

ret = iommu_map_sg(domain, iova, sg, nents, prot, GFP_ATOMIC);
...
return __finalise_sg(dev, sg, nents, iova);
```

`iommu_map()` 再进入 driver 的 `map_pages`：

```c
int iommu_map(struct iommu_domain *domain, unsigned long iova,
              phys_addr_t paddr, size_t size, int prot, gfp_t gfp)
{
    ret = iommu_map_nosync(domain, iova, paddr, size, prot, gfp);
    if (ret)
        return ret;

    ret = iommu_sync_map(domain, iova, size);
    if (ret)
        iommu_unmap(domain, iova, size);
    return ret;
}
```

### 12.2 Linux guest passthrough：DMA address 通常是 GPA/IPA

如果 guest 的设备直通由 hypervisor 管理，guest 驱动可能拿不到真实物理地址，
而是拿到 guest physical address。设备把 GPA/IPA 发给 SMMU，由 hypervisor
配置的 stage-2 翻译到 HPA。

### 12.3 Xen Dom0：Dom0 看不到 Xen SMMU

在 Xen-managed SMMU 路径中，Dom0 的设备树会移除 SMMU 和 IOMMU binding。
Dom0 Linux 通常不会运行本文的 Linux SMMU driver。

这时：

- Dom0 设备驱动把 DMA address 当作 Dom0 IPA；
- Xen SMMU stage-2 使用该 Dom0 P2M；
- P2M 更新由 Xen 负责；
- Dom0 Linux 不需要 `iommu_map()`。

这就是 Linux SMMU 代码和 Xen SMMU 代码不能简单理解成“同一个系统里两层
driver 同时工作”的原因。

## 13. 启动和所有权对比

### 13.1 Linux host

```text
Linux boot
  -> probe SMMU platform device
  -> discover SMMU features
  -> register iommu_device + iommu_ops
  -> bus/device probe
  -> of_iommu_configure()
  -> iommu_fwspec
  -> iommu_probe_device()
  -> SMMU probe_device()
  -> allocate IOMMU group
  -> allocate/attach default IOMMU domain
  -> install IOMMU DMA ops
  -> device driver DMA map/unmap
```

Linux 通用 IOMMU core 的入口：

- [`iommu_init_device()`](../../linux/drivers/iommu/iommu.c:469)
- [`iommu_probe_device()`](../../linux/drivers/iommu/iommu.c:709)
- [`iommu_setup_dma_ops()`](../../linux/drivers/iommu/dma-iommu.c:2166)

Linux `iommu_init_device()` 的关键部分：

```c
ops = iommu_fwspec_ops(dev->iommu->fwspec);
iommu_dev = ops->probe_device(dev);
if (IS_ERR(iommu_dev))
    return PTR_ERR(iommu_dev);

dev->iommu->iommu_dev = iommu_dev;
ret = iommu_device_link(iommu_dev, dev);

group = ops->device_group(dev);
dev->iommu_group = group;

dev->iommu->max_pasids = dev_iommu_get_max_pasids(dev);
```

这解释了为什么 Linux IOMMU driver 的 `probe_device()` 负责建立硬件
master 关系，而通用 IOMMU core 负责 group、默认 domain 和 DMA ops。

### 13.2 Xen

```text
Xen boot
  -> iommu_setup()
  -> probe SMMU
  -> register Xen iommu_ops
  -> create domain
  -> iommu_domain_init()
  -> p2m_init()
  -> parse device tree / ownership
  -> iommu_add_dt_device()
  -> iommu_assign_dt_device()
  -> SMMU attach to domain P2M
```

Linux 的 domain 由 IOMMU core、DMA API、VFIO、IOMMUFD 使用；Xen 的
domain 是 Xen 调度和 P2M 的基本单位。

Xen 把设备绑定到 domain 的框架代码
（[device_tree.c](../../xen/xen/drivers/passthrough/device_tree.c:29)）：

```c
int iommu_assign_dt_device(struct domain *d, struct dt_device_node *dev)
{
    int rc = -EBUSY;
    struct domain_iommu *hd = dom_iommu(d);

    if ( !is_iommu_enabled(d) )
        return -EINVAL;
    if ( !dt_device_is_protected(dev) )
        return -EINVAL;

    spin_lock(&dtdevs_lock);
    if ( !list_empty(&dev->domain_list) )
        goto fail;

    rc = hd->platform_ops->assign_device(d, 0, dt_to_dev(dev), 0);
    if ( rc )
        goto fail;

    list_add(&dev->domain_list, &hd->dt_devices);
    dt_device_set_used_by(dev, d->domain_id);

fail:
    spin_unlock(&dtdevs_lock);
    return rc;
}
```

这段代码清楚地表达了 Xen 的三层关系：

```text
Host Device Tree device
  -> domain_iommu(d)->platform_ops
  -> SMMU driver assign_device
  -> device 记录到该 Xen domain
```

## 14. 并排源码映射

| SMMU 原理 | Linux | Xen |
| --- | --- | --- |
| SMMU 探测 | `arm_smmu_device_probe()` | `arm_smmu_dt_init()` / `arm_smmu_device_probe()` |
| 能力发现 | `arm_smmu_device_cfg_probe()` / `arm_smmu_device_hw_probe()` | 对应 driver 的 cfg/hw probe |
| DT/ACPI ID 解析 | `of_iommu_configure()` | `iommu_add_dt_device()` |
| device fwspec | Linux `struct iommu_fwspec` | Xen `struct iommu_fwspec` |
| master 登记 | `arm_smmu_probe_device()` / `arm_smmu_add_device()` | `iommu_add_dt_device()` + driver `add_device` |
| SMMUv2 SMR/S2CR | `arm_smmu_master_alloc_smes()` | `arm_smmu_dt_add_device_generic()` |
| SMMUv2 context bank | `arm_smmu_init_context_bank()` | `arm_smmu_init_context_bank()` |
| SMMUv2 attach | `arm_smmu_attach_dev()` | `arm_smmu_assign_dev()` |
| SMMUv3 Stream Table | `arm_smmu_init_strtab()` | `arm_smmu_init_strtab()` |
| SMMUv3 STE | `arm_smmu_make_s2_domain_ste()` | `arm_smmu_write_strtab_ent()` |
| domain finalise | `arm_smmu_domain_finalise()` | SMMUv2 context 初始化 / SMMUv3 domain finalise |
| mapping | `io-pgtable-arm.c` | Xen P2M |
| map 回调 | `arm_smmu_map_pages()` | `arm_iommu_map_page()` |
| unmap | `arm_smmu_unmap_pages()` | `arm_iommu_unmap_page()` |
| TLB invalidate | SMMU driver `iotlb_sync()` + CMDQ | P2M 更新触发 `iommu_iotlb_flush()` |
| fault | context fault / evtq / priq | context fault / evtq / priq |
| ATS | Linux PCI ATS + ATC invalidate | 代码有限，文件头标注非完整支持 |
| PASID/SVA | Linux CD + SVA + IOPF | 当前主路径不支持完整 SVA |

## 15. 同一硬件例子的两种运行方式

假设设备 StreamID 是 `0x1234`，DMA 地址是 `0x80000000`。

### 15.1 Linux host

```text
Linux driver
  -> dma_map_sg(buffer)
  -> allocate IOVA 0x80000000
  -> iommu_map(IOVA=0x80000000, PA=buffer)
  -> io-pgtable writes IOVA -> PA
  -> SMMUv3 STE for SID 0x1234 points to page table
  -> device sends 0x80000000
  -> SMMU looks up SID 0x1234
  -> page walk
  -> PA
```

### 15.2 Xen Dom0

```text
Dom0 device driver
  -> uses IPA 0x80000000 for DMA buffer
  -> device sends 0x80000000
  -> SMMU looks up SID 0x1234
  -> STE points to Dom0 P2M
  -> P2M translates IPA 0x80000000
  -> PA
```

Xen 中通常没有 Dom0 Linux `io-pgtable` 这一步，因为 SMMU 查看的是 Xen
P2M。

## 16. TLB、ATC 和一致性

### 16.1 SMMU TLB

SMMU 会缓存地址翻译。页表 entry 被修改或删除后，必须先失效旧 translation，
再允许新配置使用。

典型顺序：

```text
更新页表
  -> DSB/内存屏障保证新表可见
  -> SMMU TLBI
  -> CMDQ SYNC 或 SMMUv2 TLB sync
  -> 可以复用旧地址或释放旧页
```

Linux SMMUv3 的 TLBI 构造和提交：

```c
static void arm_smmu_iotlb_sync(struct iommu_domain *domain,
                                struct iommu_iotlb_gather *gather)
{
    struct arm_smmu_domain *smmu_domain = to_smmu_domain(domain);
    struct arm_smmu_tlbi tlbi = {
        .iova       = gather->start,
        .size       = gather->end - gather->start + 1,
        .iopte_size = gather->pgsize,
        .leaf_only  = true,
    };

    if (!gather->pgsize)
        return;

    arm_smmu_domain_tlbi(&tlbi, smmu_domain);
}
```

最终 TLBI 以 `CMDQ_OP_TLBI_*` 命令进入 SMMU command queue。

### 16.2 PCIe ATC

ATS 设备有自己的 ATC。SMMU TLB 失效不等于 ATC 失效。需要：

```text
更新页表
  -> invalidate SMMU TLB
  -> invalidate device ATC
  -> 保证 device 不再使用旧翻译
```

Linux SMMUv3 用 ATC invalidation 命令完成这部分；Xen 的 ATS 支持非常有限。

Linux 构造 ATC invalidation 的关键代码：

```c
static struct arm_smmu_cmd
arm_smmu_atc_inv_to_cmd(u32 sid, int ssid,
                        struct arm_smmu_tlbi *tlbi)
{
    size_t inval_grain_shift = 12;
    unsigned long page_start = tlbi->iova >> inval_grain_shift;
    unsigned long page_end =
        (tlbi->iova + tlbi->size - 1) >> inval_grain_shift;
    size_t log2_span = fls_long(page_start ^ page_end);
    size_t span_mask = (1ULL << log2_span) - 1;

    page_start &= ~span_mask;
    return arm_smmu_make_cmd_atc_inv(
        sid, ssid, page_start << inval_grain_shift, log2_span);
}
```

设备 ATS 的开关在 STE 的 `EATS` 字段和 PCI ATS capability 之间必须保持
一致。

### 16.3 Page table walk coherence

`dma-coherent` 或硬件 `COHERENT_WALK` 只表示 SMMU 读页表时与 CPU
cache 一致，不表示设备发出的 DMA 数据与 CPU cache 一致。

Linux 用 `coherent_walk`、DMA API 和 `dma_sync_*` 处理不同层次；Xen 在
非 coherent walk 平台上会清页表页到 PoC 或使用对应 cache maintenance。

## 17. Fault 的定位思路

遇到 SMMU fault 时，先判断失败在哪一层：

| 现象 | 首先检查 |
| --- | --- |
| 设备完全无法 DMA | SID 是否进入 TRANS；STE/S2CR 是否有效 |
| 合法 IOVA 仍然 fault | 页表是否 map；翻译层是否匹配 |
| unmap 后仍可访问 | SMMU TLB/ATC 是否正确失效 |
| 多设备异常串扰 | StreamID/MASK 是否重叠；PCI RID alias |
| stage-1/path 不匹配 | Linux domain 是 S1、S2 还是 nested |
| Xen Dom0 DMA fault | Dom0 DT 是否隐藏 SMMU；P2M 是否有 IPA 映射 |
| PRI request | 是否有 IOPF/SVA/PRI handler |
| 设备报告 UR/AER | ATS 配置和 ENABLE/EATS 顺序 |

Linux SMMUv2 的 fault 日志来自 context FSR/FAR/FSYNR 和 CBFRSYNRA；
Linux SMMUv3 的 fault 来自 EVTQ 事件。Xen 的日志路径可以在对应 fault
handler、EVTQ tasklet 和 PRIQ tasklet 中追踪。

## 18. 需要特别区分的实现差异

### 18.1 Linux 不是 Xen 的 Dom0 SMMU driver

Linux SMMU 代码主要描述 Linux host 自己拥有 SMMU 时的行为。Xen-managed
Dom0 passthrough 中，SMMU 被 Xen 隐藏，Dom0 Linux 通常不运行这段代码。

### 18.2 Xen 的 IOMMU domain 不只是 Linux domain

Xen 的 IOMMU translation domain 绑定到 Xen `struct domain` 和该 domain 的
P2M。Linux 的 `struct iommu_domain` 是 IOMMU core 抽象，可能对应 DMA
domain、VFIO container、IOMMUFD HWPT 或 SVA address space。

### 18.3 SMMUv2 的 SMR/S2CR 与 SMMUv3 的 STE 是层级替代关系

两者都完成“StreamID -> translation context”的查找，但：

- SMMUv2 用全局寄存器表；
- SMMUv3 用内存 Stream Table；
- SMMUv3 的配置修改需要 CMDQ cache invalidation；
- SMMUv3 的 stage-1 使用 CD table，支持 PASID。

### 18.4 Stage-2 共享 P2M 只属于 Xen 当前实现

Linux SMMUv2/v3 的 stage-2 domain 可以由 io-pgtable 建独立页表，也可以由
KVM/IOMMUFD 使用 stage-2 parent；但其代码结构和 Xen 直接以 P2M root 作为
TTBR/VTTBR 的实现不同。

## 19. 推荐阅读顺序

### 第一遍：先读硬件模型

1. 本文第 3 至第 8 节；
2. SMMUv2 SMR/S2CR/context bank；
3. SMMUv3 Stream Table/STE/CD/VTCR/VTTBR；
4. page table、fault、TLB、ATS/PRI。

### 第二遍：Linux host

1. [`of_iommu_configure()`](../../linux/drivers/iommu/of_iommu.c:114)
2. [`iommu_probe_device()`](../../linux/drivers/iommu/iommu.c:709)
3. SMMUv2 `arm_smmu_probe_device()` / `arm_smmu_attach_dev()`
4. SMMUv3 `arm_smmu_probe_device()` / `arm_smmu_attach_dev()`
5. [`io-pgtable-arm.c`](../../linux/drivers/iommu/io-pgtable-arm.c:456)
6. [`iommu_dma_map_sg()`](../../linux/drivers/iommu/dma-iommu.c:1413)

### 第三遍：Xen passthrough

1. [`iommu_add_dt_device()`](../../xen/xen/drivers/passthrough/device_tree.c:258)
2. [`iommu_assign_dt_device()`](../../xen/xen/drivers/passthrough/device_tree.c:29)
3. SMMUv2 `arm_smmu_assign_dev()`
4. SMMUv3 `arm_smmu_assign_dev()`
5. Xen P2M 更新和 IOTLB flush

### 第四遍：高级功能

1. Linux SMMUv3 ATS/PASID/SVA/IOPF；
2. VFIO/IOMMUFD 如何创建和使用 IOMMU domain；
3. KVM nested stage-1 + stage-2；
4. fault queue 和 page response；
5. ATC/SMMU/GPU TLB 的失效顺序。

## 20. 读完后应能回答

1. SMMU 为什么需要 StreamID，而不能只用 DMA 地址？
2. SMMUv2 的 SMR、S2CR 和 context bank 各自属于哪一级查找？
3. SMMUv3 的 STE 的 `CONFIG`、`S2VMID`、`VTCR`、`S2TTB` 各是什么？
4. Linux 的 `io-pgtable` 与 SMMU driver 的职责边界是什么？
5. 为什么 Xen 不需要调用 `io-pgtable` 来建立自己的 stage-2？
6. Linux DMA API 的 IOVA 和 Xen passthrough 的 IPA 有什么区别？
7. 为什么 Dom0 看不到 SMMU，设备 DMA 仍然经过 SMMU？
8. 为什么修改页表后必须失效 SMMU TLB？启用 ATS 时还需要做什么？
9. SMMUv3 的 CMDQ 和 EVTQ 分别解决什么问题？
10. Linux 的 stage-1、stage-2 和 Xen 的 stage-2-only 有何实现差异？
11. SMMU fault、GPU page fault 和 CPU page fault 为什么不是同一层？
12. Xen 的 SMMUv3 ATS/PASID 代码为什么不能直接按 Linux SVA 能力理解？

## 21. 源码总索引

Linux：

- [of_iommu.c](../../linux/drivers/iommu/of_iommu.c:22)
- [iommu.c](../../linux/drivers/iommu/iommu.c:469)
- [dma-iommu.c](../../linux/drivers/iommu/dma-iommu.c:1413)
- [io-pgtable-arm.c](../../linux/drivers/iommu/io-pgtable-arm.c:456)
- [arm-smmu.c](../../linux/drivers/iommu/arm/arm-smmu/arm-smmu.c:675)
- [arm-smmu-v3.c](../../linux/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c:3061)

Xen：

- [ARM IOMMU framework](../../xen/xen/drivers/passthrough/arm/iommu.c:51)
- [DT device assignment](../../xen/xen/drivers/passthrough/device_tree.c:258)
- [ARM iommu_fwspec](../../xen/xen/drivers/passthrough/arm/iommu_fwspec.c:32)
- [SMMUv2](../../xen/xen/drivers/passthrough/arm/smmu.c:1157)
- [SMMUv3](../../xen/xen/drivers/passthrough/arm/smmu-v3.c:2477)
- [Xen P2M](../../xen/xen/arch/arm/mmu/p2m.c:1427)

## 22. 本轮边界

本文没有验证：

- 具体 SoC SMMU 的 ID 寄存器、StreamID 和 IRQ 连线；
- 具体 PCIe host bridge 的 `iommu-map` 是否完整；
- ATS/PASID/PRI 在目标硬件和软件组合上的可用性；
- Linux IOMMU domain 类型和实际 DMA ops 的运行时选择；
- Xen Dom0 P2M 是否与设备 DMA 地址完全匹配；
- SMMU fault、ATC invalidation、GPU TLB 和 cache maintenance 的真实时序；
- 性能、热插拔、reset、迁移和安全攻击面。

这些必须在具体平台、固件、Linux、Xen 和设备驱动组合下通过运行证据确认。
