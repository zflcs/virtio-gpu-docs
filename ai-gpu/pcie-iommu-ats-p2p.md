# PCIe、IOMMU、ATS/PASID 与 GPU P2P

## 1. 文档定位

GPU 虚拟化不能只理解 GPU MMU。一个 device DMA 还可能经过：

- PCIe Requester ID；
- IOMMU stage-1/stage-2 翻译；
- ATS 地址翻译缓存；
- PASID 子地址空间；
- PRI page request；
- ACS 和 P2P 路径；
- DMA API cache maintenance/SWIOTLB。

本文把这些基础设施与 virtio-gpu、VFIO、SR-IOV、GPU SVM 和 AI 多卡通信连接起来。

源码基线：Linux `6b4b543fa18757297ac756a6fe6e64f506b7844a`。

本文不运行 IOMMU/VFIO/P2P/PASID 实验。

## 2. 地址链

### 2.1 CPU/GPU/IOMMU

```text
Guest CPU VA
  -> Guest CPU page table
  -> GPA
  -> stage-2 / EPT/NPT
  -> HPA

GPU VA
  -> GPU page table
  -> device DMA address / IOVA/GPA
  -> IOMMU
  -> HPA
```

两张表可能都在 Guest、Host 或一边一层。

### 2.2 Guest 内 vIOMMU

支持 nested translation 时可能出现：

```text
device IOVA
  -> guest stage-1 (GIOVA -> GPA)
  -> host stage-2 (GPA -> HPA)
```

Guest driver 管理 stage-1；Host/VMM 管理 stage-2。硬件是否支持两层和 cache
invalidate 协议取决于 IOMMU。

## 3. PCIe 身份与 Translation

### 3.1 Requester ID

PCIe transaction 带 Requester ID（通常 Bus/Device/Function）。IOMMU 用 RID
选择对应 device/context translation。

问题：

- VF 是否有可区分 RID；
- bridge 后 transaction 是否保持正确 RID；
- ATS/PRI/PASID capability 是否对目标 function 可用；
- VMM 是否把 guest-visible RID 映射到正确 Host RID。

### 3.2 ATS

PCIe ATS 允许设备缓存地址翻译（ATC），减少对 IOMMU 的重复请求。

关键点：

- ATS translation request 和 invalidation 必须与 IOMMU page-table 更新同步；
- device ATC 不等于 GPU TLB；
- IOMMU invalidation 完成后仍可能需要 GPU TLB invalidate；
- Guest/Host nested translation 下，两层 invalidation 都要正确。

Xe 源码中 `PAT_ATS` 等路径说明 GPU memory/ATS 属性是独立硬件配置，不能只看
Host IOMMU。

### 3.3 PASID

PCIe PASID 让一个 device function 携带 process/address-space ID，以访问多个
stage-1 address spaces。

GPU/KFD 使用 PASID 关联：

- process `mm`；
- GPU fault；
- event interrupt；
- IOMMU binding；
- queue/VM。

`kfd_process_device_init_vm()` 的注释明确说明 process 的 PASID 要 bind 到设备，
process exit/device remove 时 unbind。

### 3.4 PRI

PCIe PRI 让 device page table miss 请求 Host/VM 解析，而不是直接失败。

IOMMUFD 的 FAULT object 为 HWPT 提供用户态 fault queue，可用于处理 PRI
fault。它与 GPU page fault 可能是不同层：

```text
GPU VA miss -> GPU page fault
device DMA IOVA miss -> IOMMU/PRI fault
```

## 4. IOMMU 的三类作用

### 4.1 Translation

把 device DMA 地址翻译成 Host physical page。

### 4.2 Isolation

把 device 限制在允许的 page/IOVA 范围，不能被共享 GPU 的另一租户控制。

### 4.3 Translation cache/fault

维护 TLB/ATC/PRI/invalidation。安全隔离不只是 map 成功，还要在 unmap 后
停止旧翻译继续命中。

## 5. VFIO、IOAS 和 IOMMU group

### 5.1 VFIO group

[VFIO 文档](../../linux/Documentation/driver-api/vfio.rst:35) 说明：

- DMA 是直通最关键的完整性风险；
- IOMMU 可提供 device isolation；
- 设备 enclosure/bridge/ACS 可能让实际隔离粒度大于单个 device；
- IOMMU group 是 VFIO ownership 的最小单位。

一个 PCI function 能单独看到，不代表它能单独安全 assign；同组设备的
multifunction backdoor 或 non-ACS bridge 可能绕过 Host 预期路径。

### 5.2 IOMMUFD

[IOMMUFD 用户 API](../../linux/Documentation/userspace-api/iommufd.rst:1)
提供：

- `IOAS`：IOVA-to-memory address space；
- `DEVICE`：绑定到 iommufd 的设备；
- `HWPT_PAGING`：kernel-managed stage-1/unmanaged 或 stage-2 nesting parent；
- `HWPT_NESTED`：guest-managed nested stage-1；
- `FAULT`：PRI/fault event queue；
- `vIOMMU`：物理 IOMMU virtualization slice；
- `vDEVICE`：device 在 vIOMMU 上的 virtual identity；
- `vEVENTQ`：translation/hardware events；
- `HW_QUEUE`：硬件加速 virtual queue。

### 5.3 Ownership

IOMMUFD 文档强调 device 必须先通过 driver claim DMA ownership，且在绑定完成
前 driver 不应访问设备。物理设备不能同时被多个 DMA owner 使用。

这是“打开 `/dev` 设备”和“安全拥有 DMA 路径”之间的差异。

## 6. IOMMUFD nested translation

```text
IOAS
  -> HWPT_PAGING (stage-2 parent in VM use)
  -> HWPT_NESTED (guest-managed stage-1)
  -> device
```

关键顺序：

1. VMM 建 IOAS/stage-2；
2. VMM 分配 nesting-parent HWPT；
3. Guest 管理 stage-1 page table；
4. VMM 通过 HWPT_NESTED 关联；
5. 设备 attach；
6. write 后执行嵌套 page-table invalidation；
7. fault/event 按配置返回用户态。

IOMMUFD 文档指出，不兼容的 parent/nested domain 组合必须被 driver 拒绝。

## 7. IOMMUFD 版本边界

固定文档的 Core Concept 写着：device 最多 attach 一个 IOAS，
“no support of PASID yet”。但同一固定 Linux 的
[`iommufd.h`](../../linux/include/uapi/linux/iommufd.h:417) 已定义
`IOMMU_HWPT_ALLOC_PASID` 和 PASID hardware info。

这说明：

- 文档概述可能慢于 UAPI；
- flag 存在不等于所有 IOMMU driver/CPU 支持；
- 仍需检查具体 `iommu_hw_info`、driver op 和 attach 流程；
- 不能仅凭 header 断言生产可用。

这也是证据登记册为何区分 UAPI/implemented/released。

## 8. DMA coherence 与 IOMMU

IOMMU 负责地址翻译和访问控制，不负责使 CPU/GPU cache 自动一致。

`dma_map_page()`/`dma_map_sgtable()` 还负责：

- 生成 DMA address；
- 按方向做 cache sync；
- 在不连续/SWIOTLB 平台使用 bounce buffer；
- 维护 mapping lifetime。

因此：

```text
IOMMU mapped
  != CPU/GPU coherent
  != GPU PTE correct
```

详见 [GPU 显存与一致性](gpu-memory-semantics.md)。

## 9. Virtio-GPU 中的 IOMMU

### 9.1 API remoting

Guest virtio command/resource 到 QEMU：

- Guest 发 GPA/IOVA 描述；
- QEMU device DMA address space 解析；
- Host renderer 得到 iovec/Host VA/dma-buf；
- Host renderer 自己的 GPU KMD 再建立 GPU/IOMMU 映射。

Guest 资源在 renderer 中可见，不等于 Guest 物理 GPU 有可直接写 Host HPA
的 DMA capability。

### 9.2 Native DRM context

Guest resource、dma-buf 和 Host native BO 经过 resource table/import/export。
Host native driver 管理 Host GPU VM；vfio/IOMMU 仍保护 renderer 和 Host
device 之间的 system memory。

### 9.3 VFIO/SR-IOV

Guest 原生 GPU driver 配置 Guest GPU VM。GPU DMA 仍需 Host IOMMU：

```text
Guest GPU VA -> GPU PTE -> GPA/IOVA -> Host IOMMU -> HPA
```

VF 的 SR-IOV 配额不能替代 IOMMU；IOMMU 也不能替代 GPU command/firmware
隔离。

## 10. AMD KFD 与 PASID

KFD process-device VM：

- GPU VM 与 process/PASID 关联；
- IOMMU 需要 bind `pasid->mm`；
- GPU fault/event interrupt 按 PASID 找 process；
- process exit/device remove 要 unbind。

GPU SVM range 的 notifier、page fault、migration 和 GPU TLB invalidate 是 GPU
页表层；Host IOMMU 仍约束 GPU 对 system memory 的 DMA。

## 11. Intel Xe 与 ATS/PASID

Xe LRC 保存 `CS_CTX_SYS_PASID`；PAT 中存在 ATS entry；SVM stats 记录
GPU page fault/TLB invalidation。它们说明：

- context 携带 PASID；
- GPU memory type 有 ATS/PASID cacheability 配置；
- GPU TLB invalidation 与 IOMMU invalidation 是独立统计/路径。

检查 SR-IOV VF 时必须同时确认 VF 的 IOMMU、ATS/PASID 暴露与 Guest Xe
driver 能力。

## 12. P2P 与 ACS

NVLink、XGMI、UALink NPA、RDMA/GDR 和 CXL 的更高层比较见
[AI GPU 互连与虚拟化 Fabric](ai-interconnect-fabric-virtualization.md)。

### 12.1 P2P 路径

GPU/GPU 或 GPU/NIC 可能通过：

- PCIe P2P；
- NVLink；
- XGMI/UALink；
- CXL；
- Host memory bounce；
- network RDMA。

路径选择由 topology、IOMMU、ACS、driver/provider 决定。

### 12.2 ACS

PCIe ACS 可以阻止或控制 peer-to-peer transaction 在 switch/bridge 间绕过
Root Complex/IOMMU。

如果 ACS 配置让 P2P 不可达，collective 可能：

- fallback 到 Host memory；
- 选择网络路径；
- hang/初始化失败；
- 性能显著下降。

IOMMU group 和 P2P isolation 必须一起看。

### 12.3 多 VM

跨 VM P2P 通常比同 VM 多 VF 更复杂：

- 需要设备共享机制；
- IOMMU domain 跨越；
- 安全边界；
- topology 可见性；
- NCCL/RCCL provider 支持。

不能假设 PCIe switch 有物理连通就安全可通信。

## 13. Failure matrix

| 症状 | 优先检查 |
| --- | --- |
| VFIO 绑定失败 | group 其他 device、driver bound、IOAS |
| Device DMA fault | IOAS map、IOVA range、RID/context |
| GPU fault 但 IOMMU 无 fault | GPU PTE/GPU VA 层 |
| IOMMU fault 但 GPU 正常工作 | DMA/system memory mapping |
| unmap 后仍写旧内存 | GPU/IOMMU TLB/ATC invalidation 顺序 |
| P2P 不可用 | ACS、IOMMU、topology、support |
| PASID fault 错 process | PASID/mm bind、context、fault queue |
| 性能周期性下降 | ATS/ATC invalidation、IOMMU TLB flush |
| confidential path 明文 | SWIOTLB/bounce/Host renderer |

## 14. 验证清单

1. Device requester ID 是否唯一/正确？
2. IOMMU group 是否包含 bridge/multifunction backdoor？
3. IOAS 只映射预期 IOVA 范围？
4. Map/unmap 后 IOMMU TLB/ATC 是否 invalidated？
5. GPU PTE/TLB 是否也同样 invalidated？
6. PASID 是否绑定正确 process/mm？
7. fault 在 GPU、IOMMU/PRI、CPU 哪一层？
8. ATS 是否启用，invalidation 谁负责？
9. P2P 是否绕过预期 IOMMU 边界？
10. DMA 是否需要 SWIOTLB/bounce？
11. 设备 reset 后旧 DMA/translation 是否清除？
12. VMM/vIOMMU/RID 映射是否保持 Guest/Host 语义？

## 15. 检查题

1. GPU MMU 和 Host IOMMU 的职责有何不同？
2. IOVA、GPA、HPA 和 GPU VA 如何转换？
3. 为什么 IOMMU group 可能包含多个 PCI 设备？
4. ACS 如何影响 P2P 和 VFIO 隔离？
5. ATS 与 GPU TLB 是否同一 cache？
6. PASID 如何把 DMA/fault 关联到 process？
7. PRI fault 与 GPU page fault 有何区别？
8. IOAS、HWPT_PAGING、HWPT_NESTED 和 vIOMMU 如何组成 nested translation？
9. 为什么 fixed IOMMUFD 文档与 `iommufd.h` 的 PASID 表述需要版本审计？
10. Virtio-GPU API remoting 为什么仍依赖 QEMU/renderer 的 DMA 路径安全？
11. VF/SR-IOV quota 为什么不能替代 IOMMU？
12. 为什么 IOMMU map 成功不代表 CPU/GPU coherent？

ATS/PASID/PRI capability、IOMMU SVA bind、IOPF 和 unbind/flush 的逐函数源码见
[PCIe ATS、PASID、PRI 与 IOMMU SVA 源码跟踪](ats-pasid-pri-source-trace.md)。

## 16. 与其他专题

- 显存与地址：  
  [GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)
- 安全与故障域：  
  [GPU 虚拟化的隔离、安全与故障域](gpu-virtualization-security.md)
- 平台与 VFIO：  
  [GPU 虚拟化平台比较](../virtio-gpu/platform-comparison.md)
- AMD PASID/KFD：  
  [AMD KFD/ROCr 源码跟踪](amdkfd-source-trace.md)
- Intel Xe ATS/PASID：  
  [Intel Xe GPU 虚拟化与 SR-IOV](intel-xe-sriov.md)
- Collective/P2P：  
  [NCCL/RCCL Collective 源码跟踪](nccl-rccl-source-trace.md)
- ATS/PASID/PRI 源码：  
  [IOMMU SVA 源码跟踪](ats-pasid-pri-source-trace.md)
