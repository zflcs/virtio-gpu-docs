# NVIDIA Grace GPU VFIO Variant 源码跟踪

## 1. 范围与证据边界

对应通用机制：

- [Linux VFIO/IOMMUFD/SR-IOV/mdev 源码](vfio-iommufd-sriov-source-trace.md)
- [QEMU VFIO 源码](qemu-vfio-source-trace.md)
- [NVIDIA CUDA、MIG 与 vGPU 架构边界](nvidia-cuda-vgpu.md)

本文跟踪 Linux 的
`drivers/vfio/pci/nvgrace-gpu/main.c`。这是一个面向 NVIDIA Grace Hopper
Superchip GPU 的 VFIO PCI variant driver，不是 virtio-gpu renderer，也不是
API remoting。

固定源码：

```text
Linux 6b4b543fa18757297ac756a6fe6e64f506b7844a
drivers/vfio/pci/nvgrace-gpu/main.c
include/linux/vfio_pci_core.h
```

源码可证明：

- PCI device ID 和 probe 条件；
- GPU device memory 如何从 ACPI/CXL 信息中取得；
- fake 64 位 BAR、config emulation 和 mmap；
- reset 后等待 HBM/C2C/CXL ready 的顺序；
- 非 cacheable resmem workaround；
- dma-buf physical range/P2P provider 选择。

源码不能单独证明：

- 某台 Grace 系统的实际内存训练、C2C link 或 VM 运行成功；
- Guest NVIDIA/CUDA driver 的全部兼容性；
- confidential VM、attestation 或 encrypted GPU memory；
- 所有 GB 系统的内部 firmware/partition 机制；
- 性能、peer topology 或迁移能力。

本文不运行 VM、GPU workload、reset、CXL 或 P2P 实验。

## 2. 为什么这不是普通 `vfio-pci`

普通 NVIDIA GPU 直通通常让 Guest 通过 Host GPU BAR 访问设备寄存器。Grace
平台的 GPU device memory 还可以通过 CPU-GPU coherent 互连成为 Host 可寻址
物理内存，因此 QEMU 不必只暴露原始 Vendor BAR layout。

本驱动的核心思路：

```text
Host ACPI / CXL
  -> discover CPU-accessible GPU memory
  -> build usemem / optional resmem
  -> fake 64-bit device BARs
  -> VFIO mmap / read / write / dma-buf
  -> QEMU exposes region to Guest
  -> Guest NVIDIA driver manages queues and GPU execution
```

QEMU 和 VFIO 只负责“让 Guest 正确看到可访问内存和 PCI 设备”。GPU command、
CUDA context、queue、doorbell 和 fence 仍由 Guest NVIDIA driver 理解。

## 3. 支持的设备与驱动入口

固定源码的 PCI ID table 包含：

| 注释 | Device ID |
| --- | --- |
| GH200 120GB | `0x2342` |
| GH200 480GB | `0x2345` |
| GH200 SKU | `0x2348` |
| GB200 SKU | `0x2941` |
| GB300 SKU | `0x31C2` |

驱动通过 `PCI_DRIVER_OVERRIDE_DEVICE_VFIO()` 取得 variant-driver binding
优先级。`pci_driver.driver_managed_dma = true` 表示该驱动自己管理 DMA
ownership；这是 VFIO 和 IOMMUFD ownership 检查的一部分。

[`nvgrace_gpu_probe()`](../../linux/drivers/vfio/pci/nvgrace-gpu/main.c:1336)
先尝试读取 Grace 内存属性：

- 成功：使用完整的 `nvgrace_gpu_pci_ops`；
- 失败：退回通用 `nvgrace_gpu_pci_core_ops`。

因此同一个模块有两种模式：

```text
memory-aware mode
  -> fake BAR + memory mapping + ready/reset handling

generic VFIO fallback
  -> ordinary vfio-pci core behavior
```

只看模块加载成功不能判断当前实例处于哪条路径。

## 4. Grace memory 从哪里来

[`nvgrace_gpu_fetch_memory_property()`](../../linux/drivers/vfio/pci/nvgrace-gpu/main.c:1127)
读取 ACPI Device Properties：

```text
nvidia,gpu-mem-base-pa
nvidia,gpu-mem-size
```

源码检查：

- base 和 size 是否超过 `phys_addr_t`/`size_t`；
- size 是否为 0；
- C2C link 错误时 size 为 0，因此 probe 失败。

这一步说明 GPU memory 的 Host 物理基地址是平台固件提供的资源，不是 driver
任意分配出来的。地址重叠、C2C link 和 firmware 状态错误会在设备可分配前
暴露。

## 5. usemem 与 resmem

### 5.1 memory-aware layout

[`nvgrace_gpu_init_nvdev_struct()`](../../linux/drivers/vfio/pci/nvgrace-gpu/main.c:1163)
构造两个 `struct mem_region`：

| Region | VFIO BAR | 用途 | Cache |
| --- | --- | --- | --- |
| `usemem` | BAR4 + BAR5（64-bit BAR4） | VM workload 可用 GPU memory | NORMAL cached |
| `resmem` | BAR2 + BAR3（64-bit BAR2） | GH MIG workaround 所需非 cacheable memory | NORMAL_NC |

普通布局：

```text
device memory
  | usemem: usable ---------------------------------|
  | optional resmem: non-cached --------------------|
```

### 5.2 Grace Hopper MIG hardware bug

注释说明部分 Grace Hopper 系统的 VM GPU driver 为支持 MIG 需要 NORMAL_NC
memory。由于整个 device memory 被声明为 NORMAL cached，驱动从末尾 carve out
`resmem`。

代码先计划保留 `1 GiB`，再把 `usemem` 向下对齐到硬约束：

```text
MEMBLK_SIZE = 512 MiB
```

所以实际 `resmem` 可能大于初始 1 GiB 的意图值。`resmem` 的准确大小以
`memlength - round_down(memlength - 1GiB, 512MiB)` 为准。

512 MiB 不是通用 VFIO 常量，而是源码注释明确指出的 GPU firmware/VFIO/VM
driver 三方 ABI。

### 5.3 Grace Blackwell 的修复

[`nvgrace_gpu_has_mig_hw_bug()`](../../linux/drivers/vfio/pci/nvgrace-gpu/main.c:1256)
读取 NVIDIA DVSEC 的 bitmap：

```text
MIG_SUPPORTED_WITH_CACHED_RESMEM = BIT(0)
```

如果硬件支持 cached resmem，则不再划分 `resmem`，全部 memory 进入
`usemem`。这是 capability-driven workaround，不应按 GPU 产品名硬编码。

## 6. Fake BAR 的 ABI

### 6.1 Size 与 mapping 不是同一个值

[`nvgrace_gpu_ioctl_get_region_info()`](../../linux/drivers/vfio/pci/nvgrace-gpu/main.c:535)
报告的 `bar_size` 是实际 memory length 向上取到 2 的幂。

但 sparse mmap capability 只声明：

```text
offset = 0
size   = actual memlength
```

原因：PCI BAR size 通常需要 power-of-two，而实际 device memory 大小不一定。

```text
reported BAR size: [0 ................ pow2]
mappable capacity: [0 ... memlength]
                    actual device memory
```

### 6.2 Config space

真实 GPU BAR 寄存器不能直接代表 fake memory BAR，所以驱动自行模拟：

- BAR2/3 的 64-bit value；
- BAR4/5 的 64-bit value；
- 64-bit memory、prefetch type；
- BAR address 对齐和 `bar_val`。

[`nvgrace_gpu_read_config_emu()`](../../linux/drivers/vfio/pci/nvgrace-gpu/main.c:624)
先读取 core config，再用 fake value 覆盖对应 BAR dword。

[`nvgrace_gpu_write_config_emu()`](../../linux/drivers/vfio/pci/nvgrace-gpu/main.c:675)
把 Guest 写入的 BAR value 存在 driver state，而不是写到不存在的物理 BAR。

这对 Guest NVIDIA driver 很关键：它仍按标准 PCI 枚举配置 64 位 BAR，但实际
backend 是 VFIO variant。

## 7. mmap fault 路径

[`nvgrace_gpu_mmap()`](../../linux/drivers/vfio/pci/nvgrace-gpu/main.c:476)：

1. 解码 VFIO region index；
2. 检查请求是否越过 actual memory；
3. 设置 `VM_IO | VM_PFNMAP | VM_DONTEXPAND | VM_DONTDUMP`；
4. `resmem` 额外设置 `VM_ALLOW_ANY_UNCACHED` 和 write-combine；
5. 安装自定义 `vm_operations`。

[`nvgrace_gpu_vfio_pci_huge_fault()`](../../linux/drivers/vfio/pci/nvgrace-gpu/main.c:393)
将 VFIO BAR offset 转成 GPU memory PFN，并：

- 检查 huge page 对齐；
- 检查 runtime PM；
- reset 后检查 device ready；
- 在 memory lock 下调用 `vfio_pci_vmf_insert_pfn()`。

第一 page fault 可能涉及设备 ready 等待。源码还处理“等待期间又发生 reset”
的 retry，避免把可恢复状态直接转成 SIGBUS。

### 7.1 NORMAL_NC 的特殊授权

`resmem` 通过 `VM_ALLOW_ANY_UNCACHED` 告知 KVM 可以建立 NORMAL_NC 映射。
源码注释明确说该模块接受这种 mapping 的 uncontained failure 风险。

这意味着 cache attribute 不是普通性能提示，而是跨 Guest/Host/CPU/GPU 的
内存一致性契约。错误 attribute 可能造成数据破坏或 RAS event。

## 8. VFIO read/write fallback

VFIO userspace 不保证只通过 mmap 访问 region。
[`nvgrace_gpu_read()`](../../linux/drivers/vfio/pci/nvgrace-gpu/main.c:867)和
[`nvgrace_gpu_write()`](../../linux/drivers/vfio/pci/nvgrace-gpu/main.c:997)
还支持 read/write syscall 路径。

### 8.1 Kernel VA 映射

[`nvgrace_gpu_map_device_mem()`](../../linux/drivers/vfio/pci/nvgrace-gpu/main.c:717)：

- `usemem`：`memremap(..., MEMREMAP_WB)`；
- `resmem`：`ioremap_wc()`。

映射结果由 `remap_lock` 保护，并在 close 时 unmap。

### 8.2 Backing 大小不规则的语义

若实际 device memory 不是 power-of-two：

- offset 超过 reported BAR size：`-EINVAL`；
- 读取在 actual memory 之外、reported size 之内：返回 `0xFF`；
- 写入 actual memory 之外、reported size 之内：丢弃；
- 访问可以跨实际大小并在 reported size 处截断。

这是 fake BAR 必须维持的稳定 ABI。Guest driver不能把“BAR 中读到全 1”
误判为真实 device memory 内容。

### 8.3 Runtime PM

memory read/write 外层调用 `pm_runtime_resume_and_get()`。挂起状态返回错误
而不是直接访问硬件。first-fault 路径也检查 `pm_runtime_engaged` 并返回
SIGBUS。

## 9. IOEVENTFD 被禁用

[`nvgrace_gpu_ioctl()`](../../linux/drivers/vfio/pci/nvgrace-gpu/main.c:590)
明确对 `VFIO_DEVICE_IOEVENTFD` 返回 `-ENOTTY`。

这并不表示 GPU 没有 doorbell；它表示该 variant 的 fake memory BAR/config
路径不能沿用通用 vfio-pci 的 IOEVENTFD 假设。具体 Guest submission 由
NVIDIA guest driver 与真实 GPU programming interface 决定，不在这段
source 中展开。

分析与 trace 时不能默认 vfio-pci 的 MMIO doorbell fast path 一定可用。

## 10. Device ready 与 reset

### 10.1 Legacy BAR0 检查

[`nvgrace_gpu_wait_device_ready_legacy()`](../../linux/drivers/vfio/pci/nvgrace-gpu/main.c:258)
轮询 BAR0 中与以下状态相关的寄存器：

- CPU-GPU C2C link；
- HBM training。

两者都读到 `0xFF` 才 ready。轮询以 1 秒为 quantum，总 timeout 为 30 秒，
并且使用 killable sleep。

### 10.2 CXL DVSEC 检查

[`nvgrace_gpu_wait_device_ready_cxl()`](../../linux/drivers/vfio/pci/nvgrace-gpu/main.c:321)
读取 CXL DVSEC Range 1 Low：

```text
MEM_INFO_VALID
MEM_ACTIVE
MEM_ACTIVE_TIMEOUT
```

按固定源码注释的 CXL r4.0 编码：

```text
1s, 4s, 16s, 64s, 256s
```

超过编码范围会 clamp 到 256 秒。probe 路径可能因此阻塞较长时间；源码注释也
指出这可能延迟 device unbind/sysfs。

### 10.3 Reset 后首次访问

[`nvgrace_gpu_check_device_ready()`](../../linux/drivers/vfio/pci/nvgrace-gpu/main.c:358)
在 `reset_done` 时检查 memory enable 和设备 ready，成功后清除 flag。

[`nvgrace_gpu_vfio_pci_reset_done()`](../../linux/drivers/vfio/pci/nvgrace-gpu/main.c:1424)
由 AER reset completion 设置 `reset_done`。

设计动机是避免 GPU reset 后 CPU 访问未 ready memory 产生 corrected RAS
event。CXL 可等待 256 秒的检查放在 `memory_lock` 外，只在锁内做快速状态
检查，避免长时间阻塞 reset/memory 操作。

### 10.4 VFIO reset

[`nvgrace_gpu_ioctl()`](../../linux/drivers/vfio/pci/nvgrace-gpu/main.c:590)
在 `VFIO_DEVICE_RESET` 时先清空 fake BAR emulation value，再交给 core VFIO
PCI reset。

说明 fake config state 也属于设备生命周期。只 reset 真实硬件而保留 stale
`bar_val`，可能让 Guest 看到错误 BAR 地址。

## 11. Runtime PM 与 AER

`VFIO_PCI_CORE_DEFINE_CALLBACKS()` 为 variant 生成：

- runtime suspend；
- runtime resume；
- AER error detection。

这些回调转调通用 `vfio_pci_core_runtime_suspend/resume` 和
`vfio_pci_core_aer_err_detected`。

GPU 的 AER 不是普通网卡的简单 link error：

- MMIO 可能不可访问；
- HBM/C2C 状态可能变化；
- fake memory mapping 仍可能被 userspace 持有；
- reset completion 后第一次访问必须重新检查 ready。

源码提供 readiness gate，但完整 CUDA/HIP job 恢复仍需上层重建 context 和
workload。

## 12. DMA-BUF 与 P2P

[`nvgrace_get_dmabuf_phys()`](../../linux/drivers/vfio/pci/nvgrace-gpu/main.c:1022)
处理 fake BAR 的 physical ranges：

1. 找到 `usemem`/`resmem`；
2. 使用 BAR0 的 `pcim_p2pdma_provider()`；
3. 用 `vfio_pci_core_fill_phys_vec()` 填 GPU memory physical ranges；
4. 非 fake BAR 回退到 core implementation。

源码注释指出：

- `resmem` 与 BAR memory 的 P2P 属性相同；
- `usemem` 是 cacheable memory，当前没有完整方法让 cached memory 走所有
  P2P 路径；
- Grace hardware 对 SMMU 中 cache attribute 的处理使 BAR0 provider可用；
- 未来 CXL P2P 支持可能需要 CXL-specific provider。

因此：

```text
VFIO DMA-BUF feature available
  != every fake memory region has identical P2P semantics
```

AI 多 GPU peer copy、NCCL/RCCL P2P 和 RDMA GDR 仍然需要 topology 与实测证据。

## 13. Guest 生命周期

### 13.1 Probe

```text
PCI match
  -> allocate embedded vfio_pci_core_device
  -> detect CXL DVSEC
  -> wait device ready
  -> choose full/core ops
  -> fetch ACPI memory properties
  -> calculate usemem/resmem and BAR size
  -> vfio_pci_core_register_device
```

### 13.2 QEMU/VFIO open

```text
VFIO bind and IOMMU attach
  -> nvgrace_gpu_open_device
  -> vfio_pci_core_enable
  -> register PFN address spaces
  -> expose fake BAR region info
  -> Guest config enumeration
  -> Guest mmap / DMA mapping
```

### 13.3 Close

```text
unregister PFN spaces
  -> unmap memremap/ioremap
  -> close core VFIO device
```

[`nvgrace_gpu_close_device()`](../../linux/drivers/vfio/pci/nvgrace-gpu/main.c:228)
显式清理 `memaddr/ioaddr`，避免长期保留 GPU physical memory 的 kernel VA
mapping。

## 14. 与 virtio-gpu、SR-IOV、MIG 的关系

| 维度 | Grace GPU VFIO variant | virtio-gpu | SR-IOV VF |
| --- | --- | --- | --- |
| Guest device | NVIDIA PCI GPU | `1af4:1050` virtual device | PCI VF |
| Guest driver | NVIDIA native KMD/UMD | virtio-gpu KMD + capset UMD | Vendor VF driver |
| GPU command owner | Guest vendor driver | Host renderer/proxy | Guest VF driver |
| Host role | PCI/VFIO/coherence/memory mapping | virtqueue/renderer | PF + firmware/VFIO |
| Multi-VM sharing | 本驱动主要是整 GPU assignment | 多个 Guest 共享 renderer | 多个 VF |
| GPU memory exposure | fake 64-bit BAR / DMA-BUF | blob/ring/resource/transfer | VF BAR/VRAM profile |
| Reset | VFIO/AER + readiness gate | virtio/QEMU/renderer lifecycle | VF FLR/PF management |

MIG 在本驱动中仅体现为 `resmem` hardware-workaround 条件，不能据此推断该
设备已创建或分配 MIG instance。VFIO variant 也不是 NVIDIA vGPU 产品。

## 15. AI workload 的直接影响

### 15.1 大块 HBM mapping

模型权重、KV cache、梯度和优化器状态都以大块 device memory 为主。fake BAR
只映射实际可用范围，Guest allocator/query 若把 power-of-two BAR size 当作
全部可用 memory，就会越过真实范围。

### 15.2 Cache attribute

`usemem` cached 与 `resmem` NORMAL_NC 的区别会进入：

- Guest CPU access；
- GPU coherent access；
- SMMU/KVM stage-2 mapping；
- migration/dirty tracking；
- P2P provider。

这不是只影响 benchmark 的优化选项，而是正确性边界。

### 15.3 Reset 后首访问

推理服务或训练 supervisor 在 GPU reset 后立即轮询 device memory，可能触发
readiness wait/RAS workaround。需要区分：

```text
device reset 完成
  != HBM/C2C/CXL ready
  != CUDA context 已恢复
  != training job 可继续
```

### 15.4 P2P 与 collective

P2P dma-buf provider 可用只是第一层。NCCL/RCCL 还需要正确的 PCIe/NVLink/
XGMI topology、peer capability、IOMMU/ACS 和跨 VM 网络。

## 16. 失败矩阵

| 症状 | 源码检查 |
| --- | --- |
| probe 失败/超时 | ACPI memory property、C2C link、HBM training、CXL DVSEC |
| 模块使用 generic path | `nvgrace_gpu_fetch_memory_property()` 是否成功 |
| Guest 看到错误 memory 大小 | power-of-two BAR size 与 actual memlength |
| BAR mmap 越界 | 是否超过 reported/actual region |
| 读到全 1 | 是否超过 actual memory 但仍在 reported BAR 内 |
| Guest config BAR 地址异常 | `bar_val` config emulation |
| reset 后参数丢失 | `VFIO_DEVICE_RESET` 是否清 fake BAR state |
| reset 后 SIGBUS | memory enable、runtime PM、ready 检查 |
| doorbell ioctl 失败 | source 明确不支持 `VFIO_DEVICE_IOEVENTFD` |
| dma-buf/P2P 失败 | BAR0 P2PDMA provider、region physical range |
| Guest GPU driver 不兼容 | Guest vendor driver/device model/support matrix |

## 17. 已知未知项

仅读这份 kernel source 不能确定：

- Guest NVIDIA driver 如何使用这些 fake BAR；
- queue/doorbell/RM/GSP firmware 的私有协议；
- CUDA Unified Memory、SVM 或 page fault 的最佳路径；
- 每个 SKU 的 memory 大小和 C2C topology；
- confidential VM 的 encryption/attestation；
- multi-GPU P2P/NVLink 能力；
- live migration；
- 任何性能结论。

需要结合：

- 具体 GH/GB 产品文档；
- Guest NVIDIA driver 版本；
- firmware；
- QEMU/IOMMU 配置；
- 固定运行的 trace。

## 18. 推荐源码阅读顺序

1. PCI ID、module description、`pci_driver`。
2. `struct mem_region` 和 `struct nvgrace_gpu_pci_core_device`。
3. `nvgrace_gpu_probe()` 的两条 ops 路径。
4. ACPI memory property 和 `nvgrace_gpu_init_nvdev_struct()`。
5. DVSEC MIG hardware-bug detection。
6. legacy/CXL readiness。
7. region info、config emulation 和 BAR size。
8. mmap/huge fault/PFN insertion。
9. read/write fallback 和 memremap/ioremap。
10. reset_done、AER 和 runtime PM。
11. dma-buf physical vector/P2P provider。
12. 回到通用 VFIO 与 QEMU 文档检查 DMA ownership 和 Guest mapping。

## 19. 检查题

1. 这个 driver 为什么不是 virtio-gpu ROCm/Graphics renderer 的同类实现？
2. `usemem` 和 `resmem` 为什么使用不同 cache attribute？
3. 512 MiB 常量由哪几层共享？
4. 为什么 reported BAR size 和 sparse mmap size 可以不同？
5. Guest 在 actual memory 后读到全 1 的 ABI 原因是什么？
6. 为什么 config BAR write 需要 driver 自己保存？
7. reset 后的 first fault 为什么要等待 device ready？
8. CXL ready timeout 为什么需要放在 `memory_lock` 外轮询？
9. `VM_ALLOW_ANY_UNCACHED` 对 KVM stage-2 有什么含义？
10. 为什么源码禁用 `VFIO_DEVICE_IOEVENTFD`？
11. fake BAR 的 dma-buf P2P provider 为什么使用 BAR0？
12. 为什么 cached usemem 不能自动获得完整 P2P？
13. VFIO reset 成功、GPU ready 和 CUDA context 恢复为什么是三件事？
14. 看到 `resmem` 能否推断该 VM 已使用 MIG？
15. 还需要哪些 Guest driver/产品证据才能判断 PyTorch/NCCL 可用？

## 20. 与其他专题

- [NVIDIA CUDA、MIG 与 vGPU](nvidia-cuda-vgpu.md)
- [GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)
- [PCIe、IOMMU、ATS/PASID 与 GPU P2P](pcie-iommu-ats-p2p.md)
- [GPU RAS、ECC、Reset 与 AI Job 恢复](gpu-ras-ecc-reset-recovery.md)
- [Confidential GPU 与机密 AI 虚拟化](confidential-gpu-virtualization.md)
- [QEMU VFIO 源码跟踪](qemu-vfio-source-trace.md)
