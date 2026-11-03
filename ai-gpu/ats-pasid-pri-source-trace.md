# PCIe ATS、PASID、PRI 与 IOMMU SVA 源码跟踪

## 1. 目标

对应宏观文档：

- [PCIe、IOMMU、ATS/PASID 与 GPU P2P](pcie-iommu-ats-p2p.md)
- [GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)
- [VFIO、IOMMUFD、SR-IOV 与 mdev 源码](vfio-iommufd-sriov-source-trace.md)

本文跟踪 Linux 中与 GPU per-process DMA/address space 有关的四组机制：

```text
ATS
  -> device translation cache / ATC invalidation

PASID
  -> multiple process address spaces per PCI function

PRI
  -> device page request

IOMMU SVA/IOPF
  -> bind mm to device/PASID and resolve peripheral faults
```

固定源码：

```text
Linux 6b4b543fa18757297ac756a6fe6e64f506b7844a
drivers/pci/ats.c
drivers/iommu/iommu-sva.c
drivers/iommu/iommu.c
drivers/iommu/io-pgfault.c
include/linux/pci-ats.h
```

本文不运行 ATS/PASID/PRI、GPU SVM、VF 或 VM 实验。

## 2. 四层不要混用

| 层 | 对象 | 作用 |
| --- | --- | --- |
| PCIe ATS | device ATC | 缓存 translated address |
| PCIe PASID | TLP PASID prefix | 标识 process/address space |
| PCIe PRI | page request | device 请求 Host 补页 |
| IOMMU/SVA | PASID + IOMMU domain/mm | 解析命令、绑定 CPU 页表 |

GPU 还有自己的：

- GPU VA/page table；
- GPU TLB；
- GPU page fault；
- VRAM migration。

一个 GPU TLB entry、IOMMU TLB entry、PCIe ATC entry 和 CPU TLB entry 是不同
cache，失效协议也不同。

## 3. PCI ATS 初始化

[`pci_ats_init()`](../../linux/drivers/pci/ats.c:20)：

- 检查全局 ATS 禁用；
- 查找 ATS Extended Capability；
- 保存 capability offset。

[`pci_ats_supported()`](../../linux/drivers/pci/ats.c:41)：

- 无 capability -> 不支持；
- PCI device 被标记 untrusted -> 不支持；
- VF 是否支持继承 PF 的能力。

untrusted 禁用 ATS 是安全边界，而不只是性能策略。

## 4. ATS 启用

[`pci_enable_ats()`](../../linux/drivers/pci/ats.c:97)：

1. 检查 ATS supported；
2. 拒绝重复 enable；
3. 检查 page size/STU；
4. PF 设置 STU 并写 ATS Control Enable；
5. VF 仅在 PF 已使用相同 STU 时允许 enable。

[`pci_prepare_ats()`](../../linux/drivers/pci/ats.c:63)用于 PF/VF 创建前准备
相同 STU。

### 4.1 ATS STU

ATS translation request 的 untranslated address 粒度由 STU/page size 决定。
IOMMU driver 和 device capability 必须一致。

### 4.2 VF ATS

源码注释要求 PF 在 VF 创建前配置 ATS。VF 本身不能拥有不同 STU。

所以 SR-IOV 支持 PASID/ATS 时，不是只检查 VF config space；还要检查 PF
preparation 和 firmware。

## 5. ATS Invalidation

ATS 启用后 device 可缓存 translated address。IOMMU page-table unmap 之后必须：

```text
update IOMMU page table
  -> invalidate IOMMU TLB
  -> request device ATS invalidation
  -> wait ATC invalidation completion
  -> backing page can be safely reused
```

[`pci_ats_queue_depth()`](../../linux/drivers/pci/ats.c:175)查询 device 的
ATS Invalidate Request queue。

### 5.1 为什么 sequence 重要

如果释放页发生在 device ATC invalidation 完成前，GPU 仍可能使用旧 DMA
translation 写入已重新分配的内存。

这是 IOMMU unmap 安全边界中常见的缺失步骤。

### 5.2 GPU 与 IOMMU 双页表

GPU page table 改变后需要 GPU TLB invalidation；IOMMU 页表改变后需要 IOMMU
TLB + ATS invalidation。两者不能相互替代。

## 6. PASID 初始化与启用

[`pci_pasid_init()`](../../linux/drivers/pci/ats.c:435)查找 PASID capability。

[`pci_enable_pasid()`](../../linux/drivers/pci/ats.c:449)：

1. VF 只能继承 PF 的 PASID 配置；
2. 检查 EETLP prefix 或 no-TLP 支持；
3. 检查 ACS redirect/upstream forwarding 路径；
4. 检查 EXEC/PRIV feature；
5. 写 PASID Control Enable/features。

### 6.1 为什么检查 ACS

PASID 让 transaction 携带 process identity。若 PCIe switch/bridge 路径不能
正确处理请求者/PASID isolation，事务可能绕过预期边界。

## 7. PASID capability

[`pci_pasid_features()`](../../linux/drivers/pci/ats.c:551)读取：

- execute permission；
- privileged mode。

[`pci_max_pasids()`](../../linux/drivers/pci/ats.c:578)根据 capability width
返回可用 PASID 数量。

VFs share PF PASID configuration；因此 VF driver 看到的 PASID limit 不能只由
VF 自己的 capability 推断。

## 8. IOMMU SVA bind

### 8.1 mm -> PASID

[`iommu_alloc_mm_data()`](../../linux/drivers/iommu/iommu-sva.c:19)：

- 检查 CPU page table 与 IOMMU DMA 兼容；
- 每个 mm 首次 bind 分配 global PASID；
- 保存 mm、PASID、SVA domain list；
- release-store 发布 `mm->iommu_mm`。

再次 bind 同一 mm 复用 PASID，并检查 dev max_pasids。

### 8.2 Device bind

[`iommu_sva_bind_device()`](../../linux/drivers/iommu/iommu-sva.c:71)：

1. 查找 device IOMMU group；
2. 为 mm 分配/复用 PASID；
3. 查是否已有 group/PASID/domain attach；
4. 已有则增加 bind reference；
5. 否者尝试 mm 已有 SVA domains；
6. 没有兼容 domain 时分配 `IOMMU_DOMAIN_SVA`；
7. `iommu_attach_device_pasid()`；
8. 返回 `iommu_sva` handle。

### 8.3 Bind reference

同一 device/mm 可以多次 bind；handle 有 users refcount。

每次 bind 都必须在设备停止使用 PASID 后对应 unbind。

## 9. IOMMU PASID attach

[`iommu_attach_device_pasid()`](../../linux/drivers/iommu/iommu.c:3624)：

1. 要求 device 属于 IOMMU group；
2. 检查 domain 与 device IOMMU ops 兼容；
3. 拒绝 `IOMMU_NO_PASID`；
4. 检查 group 中每个 PASID-capable device 的 max_pasids；
5. 在 group PASID xarray 预留 entry；
6. 对 group 内 device 调 `set_dev_pasid()`；
7. 全部成功后才发布 domain/handle entry。

这是 group-level operation，不是只修改一个 device 的局部软件表。

### 9.1 Recovery

device recovery 期间 attach 返回 `EBUSY`，避免 PRI 在 recovery 窗口进入未知
domain。

### 9.2 Detach

[`iommu_detach_device_pasid()`](../../linux/drivers/iommu/iommu.c:3814)
遍历 group 中 PASID-capable devices，移除硬件 PASID domain 并清 xarray。

## 10. Unbind 顺序

[`iommu_sva_unbind_device()`](../../linux/drivers/iommu/iommu-sva.c:167)
最终 detach PASID/domain。

源码注释要求：

- device 不再为该 PASID 发起 transaction；
- outstanding page requests 已 flush；
- 之后才能回收 PASID。

否则新 process 复用 PASID 后可能收到旧 process 的 fault/response。

## 11. PRI 初始化

[`pci_pri_init()`](../../linux/drivers/pci/ats.c:263)查找 PRI cap，并读取
PRG Response 是否要求 PASID。

[`pci_enable_pri()`](../../linux/drivers/pci/ats.c:284)：

- VF 不拥有独立 PRI，跟随 PF；
- 检查 PRI Stopped 状态；
- 协商 outstanding request count；
- 写 allocated requests；
- enable PRI。

VF 的 PRI 与 ATS/PASID 一样依赖 PF，不能按独立 function 理解。

## 12. I/O Page Fault queue

### 12.1 Report

[`iommu_report_device_fault()`](../../linux/drivers/iommu/io-pgfault.c:214)
由 IOMMU driver 在 threaded IRQ 等上下文调用。

步骤：

1. 按 PASID/fault 查 attach handle/domain；
2. 取得 device fault params；
3. partial page requests 暂存，直到 LAST_PAGE；
4. 组成 `iopf_group`；
5. 调 domain `iopf_handler`；
6. 失败则向 device response failure/invalid。

PASID Stop Marker 不是普通 page request，IOMMU driver 应在报告前丢弃。

### 12.2 SVA fault handler

[`iommu_sva_handle_iopf()`](../../linux/drivers/iommu/iommu-sva.c:269)在
queue work 中处理：

- 取 fault 的 mm；
- 检查 PASID valid；
- 在 CPU mm 中 lookup VMA；
- 校验 read/write/exec/user permission；
- `handle_mm_fault()`；
- 向 device 返回 success/invalid。

因此 PRI fault 的解析路径实际进入 CPU `mm_struct`，不是 GPU KMD 自己补 PTE。

## 13. Fault queue 生命周期

### 13.1 Flush

[`iopf_queue_flush_dev()`](../../linux/drivers/iommu/io-pgfault.c:298)
等待 device fault workqueue 清空。IOMMU driver 在释放 PASID 前必须完成。

### 13.2 Remove device

[`iopf_queue_remove_device()`](../../linux/drivers/iommu/io-pgfault.c:464)：

- 必须先在 hardware 禁用新 PRI 并 flush queue；
- 对 outstanding PRQ 回复 invalid；
- 解除 fault param/device 关系。

### 13.3 Destroy domain

domain 只能在其 device 停止发 fault 且 pending faults flush 后释放，否则 fault
handler 可能访问已释放的 mm/domain。

## 14. VFIO/IOMMUFD 中的 PASID

VFIO device cdev 的 `VFIO_DEVICE_ATTACH_IOMMUFD_PT` 可带 PASID flag：

- IOMMUFD 提供 page table/HWPT；
- VFIO io_ops 调 device driver `pasid_attach_ioas`；
- VFIO 保持 PASID ownership/lifetime。

IOMMUFD 还提供：

- HWPT_PAGING；
- HWPT_NESTED；
- FAULT queue；
- vIOMMU/vDEVICE；
- dirty tracking。

固定 IOMMUFD 文档与 UAPI 的 PASID 描述可能随版本漂移，因此以当前 UAPI 和
driver ops 为准，不能只引用旧文档句子。

## 15. GPU 路径

### 15.1 AMD KFD

KFD process-device VM 将 process 与 GPU VM/PASID 绑定。Device interrupt 可通过
PASID/partial event ID 找到 process/event。

VFIO VF 下 Guest KFD 管理 Guest queue/VM；Host PF/firmware 管理 VF resource。
Guest IOMMU 还需处理 Guest DMA，两者不能混同。

### 15.2 Intel Xe

Xe execution context 携带 PASID，VM/BO 使用 GPU page table。SVM fault 可能先
进入 GPU fault consumer，再通过 CPU mm/HMM 或设备迁移解决。

ATS/PASID 是否暴露出售、VF 是否继承 PF capability，必须结合 SR-IOV 文档。

### 15.3 virtio-gpu

Venus/GFXStream 的 Guest UMD 不要求 Host 直接使用 Guest process PASID；Host
renderer通常有自己的 context。

ROCm native context proposal 需要代理 HSA/KFD/PASID 语义，复杂度远高于只代理
一个图形 context。

## 16. AI workload 的含义

- PyTorch/HIP/JAX 每 process context 需要稳定 per-process GPU address space；
- UVA/SVM/userptr 依赖 CPU mm notifier + GPU page table + IOMMU；
- GPU P2P 可能绕过 CPU，必须检查 ATS/ACS/IOMMU；
- copy engine/SDMA 使用相同 process PASID 还是独立 kernel descriptor，需看栈；
- 长 kernel + page fault 会阻塞 queue completion，但不一定阻止 host allocation；
- checkpoint/migration 必须停止 PASID consumers 并 flush fault queues；
- 多 VF 场景要确认 ATS/PASID 是 PF-shared 还是独立暴露。

## 17. 失败矩阵

| 症状 | 优先检查 |
| --- | --- |
| ATS enable EINVAL | cap、untrusted、STU、VF/PF |
| VF ATS enable 失败 | PF 是否 prepare 相同 STU |
| PASID enable EINVAL | TLP prefix、ACS、feature、VF/PF |
| SVA bind EBUSY | mm 已有 incompatible domain |
| SVA bind EOVERFLOW | dev max_pasids |
| PASID attach EBUSY | device recovery |
| PRI enable EBUSY | PRI stopped 状态/重复 enable |
| GPU page fault 无响应 | ATS/PASID/PRI/IOMMU fault queue |
| fault 返回 invalid | mm/VMA permission、process exit |
| unmap 后 DMA/旧页写 | GPU TLB、IOMMU TLB、ATC invalidation |
| PASID 复用收到旧 fault | stop/flush/unbind 顺序 |
| VF P2P 异常 | ACS、ATS/PASID inherited config、topology |

## 18. 审查清单

### 18.1 PCIe

- device trusted 吗？
- ATS/PASID/PRI capability 在哪一层？
- PF/VF 共享哪些 capability/config？

### 18.2 Address space

- PASID 对哪个 mm/GPU VM？
- IOMMU page table 是 stage-1、stage-2 还是 nested？
- GPU page table 与 IOMMU 页表如何对应？

### 18.3 Fault

- PRI fault 由 IOMMU/SVA、GPU KMD 还是用户态 handler 处理？
- fault group/response 完整吗？
- stop marker/flush 是否处理？

### 18.4 Teardown

- 新 transaction 何时停止？
- GPU/IOMMU/ATC TLB 何时失效？
- outstanding faults 何时 flush？
- PASID 何时安全复用？

## 19. 推荐源码顺序

1. `drivers/pci/ats.c` 的 ATS/PRI/PASID cap。
2. `drivers/iommu/iommu-sva.c` 的 mm/PASID bind。
3. `drivers/iommu/iommu.c` 的 PASID domain attach/detach。
4. `drivers/iommu/io-pgfault.c` 的 fault group/queue/response。
5. VFIO cdev PASID attach 与 IOMMUFD fault/HWPT。
6. AMD KFD与 Intel Xe 的 GPU VM/queue/PASID 使用。
7. PCIe/IOMMU/P2P 宏观文档。

## 20. 检查题

1. ATS、PASID、PRI 和 SVA 各解决什么？
2. 为什么 untrusted PCI device 不允许 ATS？
3. VF 为什么不能使用与 PF 不同的 ATS/PASID 配置？
4. ACS 与 PASID enable 有什么关系？
5. 一个 mm 何时获得 PASID，何时释放？
6. `iommu_sva_bind_device()` 为什么可以多次 bind 同一 device/mm？
7. IOMMU PASID attach 为什么是 group-level operation？
8. PRI Stop Marker 为什么可以在报告到 IOPF 前丢弃？
9. SVA IOPF 如何进入 CPU `handle_mm_fault()`？
10. 解除 PASID 前为什么必须停止 transaction 并 flush fault？
11. GPU TLB、IOMMU TLB 和 ATS ATC 为什么需要分别失效？
12. VFIO cdev/IOMMUFD 如何携带 PASID？
13. virtio-gpu 图形 remoting 与 ROCm PASID proxy 的差异？
14. 多 VF P2P 要检查哪些 PCIe/IOMMU 条件？
15. AI long-running kernel 与 IOMMU page fault 如何相互影响？

## 21. 与其他专题

- [PCIe、IOMMU、ATS/PASID 与 GPU P2P](pcie-iommu-ats-p2p.md)
- [NVIDIA Grace GPU VFIO Variant 源码](nvidia-grace-vfio-source-trace.md)
- [Intel Xe SVM、Userptr 与 GPU Page Fault 源码](intel-xe-svm-source-trace.md)
- [AMD KFD/ROCr 计算路径源码](amdkfd-source-trace.md)
- [DMA-BUF、DMA-FENCE、Reservation 与 Sync File](dma-buf-fence-source-trace.md)
- [VFIO、IOMMUFD、SR-IOV 与 mdev 源码](vfio-iommufd-sriov-source-trace.md)
