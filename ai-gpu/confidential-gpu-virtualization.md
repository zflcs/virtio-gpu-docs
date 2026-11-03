# Confidential GPU 与机密 AI 虚拟化

## 1. 目标

“CPU confidential VM 内存加密”不等于“GPU 模型、prompt、KV cache 和 HBM
机密”。GPU confidential computing 需要端到端证明：

- guest private memory；
- GPU private VRAM/HBM；
- command/kernel 路径；
- DMA private/shared transition；
- firmware/manager；
- attestation；
- telemetry/crash/migration；
- multi-GPU collective。

本文以 Linux CPU TEE public docs 为证据基座，GPU 厂商内部架构作为产品文档/
support matrix 边界，不虚构闭源实现。

本文不运行 confidential VM/GPU 实验。

## 2. 威胁模型

### 2.1 攻击者

- malicious Host operator；
- compromised Hypervisor/VMM；
- other tenant；
- Host GPU manager；
- malicious physical device/DMA；
- privileged Host service；
- supply chain/firmware；
- physical memory/cold-boot style attack（取决于 TEE 设计）。

### 2.2 资产

- model weights；
- training data；
- prompt/context；
- KV cache；
- embedding；
- gradient/optimizer；
- checkpoint；
- credential/keys；
- proprietary kernels；
- inference outputs。

### 2.3 Host 是否必须“看不见”

“Host 无法读 RAM”不等于：

- Host renderer 看不见 command；
- Host GPU driver 看不见 buffer；
- Host 不解码/转换 frame；
- Host 不会读 telemetry；
- Host 不能通过 timing 推断 workload。

必须先定义机密边界。

## 3. CPU confidential VM 基础

### 3.1 AMD SEV/SNP

[Linux SEV 文档](../../linux/Documentation/virt/kvm/x86/amd-memory-encryption.rst:1)
说明：

- VM memory 使用 VM-specific key 加密；
- ASID 关联 encryption key；
- launch/update/measure/attestation 生命周期；
- guest policy/key 交换；
- 内存内容 measurement 可供 guest owner 验证。

SEV guest 仍有 shared memory 概念；与 device/driver 交互可能需要共享页。

### 3.2 Intel TDX

[Linux TDX 文档](../../linux/Documentation/virt/kvm/x86/intel-tdx.rst:1) 说明
TDX 用 CPU-attested TDX module 保护 Guest VM。

[`tdx.rst`](../../linux/Documentation/arch/x86/tdx.rst:401) 强调：

- shared memory 可能暴露给 untrusted hypervisor；
- MMIO for virtual device 可作为 shared memory；
- device drivers 是 shared memory 的主要用户；
- DMA buffers/ioremap 可能做 private/shared conversion；
- TDX 大量使用 SWIOTLB；
- attestation 验证 TD guest/firmware/config。

### 3.3 Private 与 shared

```text
guest private memory
  encrypted, owner-only according to TEE

guest shared memory
  Host/VMM may access or provide to device
```

机密部分不能无条件与 Host 共享。

## 4. Device DMA 为什么困难

单个私密 page 不能：

- DMA 到普通 Host bounce buffer 后仍保持秘密；
- 被 Host 改写成 malformed descriptor 而不被检测；
- 同时作为 shared MMIO 和 private storage 使用。

TDX 文档通过 SWIOTLB 处理许多 DMA；SWIOTLB bounce buffer 是共享/可访问的中间页。
这可能让 Host/DMA path 看到明文。

因此 confidential GPU 必须单独证明：

- device trusted/attested；
- private DMA capability；
- shared/private page transition 的安全；
- bounce/copy 不会泄露；
- device descriptor 的完整性。

## 5. GPU confidential 的必备属性

### 5.1 Device trust

- GPU hardware/firmware 是否可信；
- Host GPU manager/firmware 能否访问 guest private data；
- device reset 是否清除 guest key/context；
- device attestation report 是否覆盖关键配置。

### 5.2 Memory

- HBM/VRAM private encryption；
- VRAM allocation/partition 是否隔离；
- GPU page table/context save 是否加密；
- legacy allocation 是否清零；
- migration/checkpoint page 是否加密。

### 5.3 Command

- kernel/command 能否被 Host GPU manager 读取；
- renderer/driver 是否解析并复制敏感 command；
- shader/module cache 是否泄漏模型/kernel；
- profile/trace 是否含 tensor 或 prompt。

### 5.4 DMA

- GPU whether trusted DMA；
- ATS/PASID identity；
- IOMMU private/shared page policy；
- bounce buffer；
- peer GPU/NIC DMA。

### 5.5 Output

- display capture/remote encoder；
- framebuffer readback；
- final output channel；
- TEE 之外的服务是否重新暴露数据。

## 6. CPU TEE + GPU 架构模式

### 6.1 CPU-only confidential VM + normal GPU

```text
confidential CPU VM
  -> ordinary virtual GPU
  -> untrusted Host renderer/driver
  -> GPU
```

CPU memory 可能加密，但：

- API remoting Host renderer 解析 command；
- Host driver 管理 GPU memory；
- Host 图片/数据会被搬运；
- 模型可能经 shared memory 离开 CPU TEE。

通常不能声称 GPU workload confidential。

### 6.2 Confidential GPU device passthrough

```text
confidential VM
  -> trusted/attested GPU device
  -> confidential HBM/command path
```

需要 CPU TEE、IOMMU、GPU firmware 和 device attestation 协同。它不是只启用
VFIO 就完成。

### 6.3 Confidential partition/vGPU

需要硬件/固件划分 private engine/context/memory。MIG/VF/vGPU 本身是资源分区，
不自动等于 confidential isolation。

### 6.4 Confidential API remoting

如果 Host renderer 能解析 API/buffer，普通 API remoting 的机密性不足。

要成立需要：

- protected Host service/TEE；
- encrypted/protected transport；
- protected GPU submission；
- protected memory sharing；
- attestation 覆盖 renderer。

仅有 TLS/socket 加密不足以证明 Host GPU runtime 不读数据。

## 7. Attestation Chain

### 7.1 目标

Guest owner 在释放模型密钥/数据前验证：

- correct CPU TEE；
- correct guest measurement；
- correct GPU device；
- correct GPU firmware/manager；
- correct confidential mode/profile；
- no debug/unsafe configuration；
- expected driver stack。

### 7.2 CPU evidence

- SEV/SNP launch measurement/attestation；
- TDX TDREPORT/quote；
- guest policy；
- VMM/secure loader identity。

### 7.3 GPU evidence

需要确认具体产品是否提供：

- GPU attestation report；
- device identity；
- firmware measurement；
- profile/config；
- nonce freshness；
- revocation；
- collective/multi-device attestation。

CPU quote 不包含 GPU measurement，除非产品建立明确组合 attestation。

### 7.4 Key release

典型流程：

```text
attest CPU+GPU
  -> verify policy/firmware/model
  -> release key to trusted guest workload
  -> decrypt/model load inside confidential domain
```

如果 Host-managed model cache 已保存明文，后续 TEE 也无意义。

## 8. Private/shared 数据路径

### 8.1 AI inference

```text
encrypted model storage
  -> guest private memory
  -> GPU private HBM
  -> kernel
  -> KV cache private HBM
  -> output policy
  -> network
```

每个“->”都可能是泄漏点。

### 8.2 Command submission

Host renderer/driver may need:

- command bytes；
- descriptors；
- addresses；
- kernel module；
- fences。

如果这些 metadata 足以推断模型或读取 tensor，也应纳入威胁模型。

### 8.3 DMA/MMIO

- shared ring/control memory may be plaintext to Host；
- private tensor memory must not use untrusted bounce；
- device MMIO can leak state/timing；
- device interrupts/fences can reveal activity。

### 8.4 Display/remote

VDI/remote inference:

- frame capture may expose prompt UI/content；
- remote encoder output is intended visible output but still sensitive；
- health/telemetry may log data.

## 9. Multi-GPU 与 Collective

Confidential training/inference must cover:

- GPU-GPU P2P private path；
- NVLink/XGMI encryption/security；
- RDMA/NIC trust；
- NCCL/RCCL communicator state；
- peer memory mapping；
- collective topology attestation；
- cross-VM network trust。

单个 confidential GPU 正常，不代表 collective 的中间 tensor 在任何 peer 上都
保持机密。

## 10. Lifecycle

### 10.1 Model load

- key release after attestation；
- encrypted storage；
- private staging；
- no Host clear-text decompression/cache。

### 10.2 Runtime

- private HBM；
- private command；
- protected queue/event；
- telemetry redaction。

### 10.3 Checkpoint

- checkpoint encrypted；
- key ownership；
- destination attestation；
- no plaintext migration staging。

### 10.4 Reset/recovery

- clear GPU keys/cache；
- zero/encrypt released HBM；
- remove context save；
- invalidate IOMMU/GPU TLB；
- attest recovered device before key release。

### 10.5 Migration

需要 source/destination CPU TEE、GPU CC mode、firmware、driver 和 attestation
兼容。destination 未 attestation 不应接收 confidential state。

## 11. 方案边界

| 方案 | 可提供的边界 | 不自动提供 |
| --- | --- | --- |
| CPU TEE + normal VFIO GPU | CPU RAM confidential | GPU HBM/command confidential |
| Confidential GPU VFIO | 可能完整 GPU TEE | 需产品/固件验证 |
| SR-IOV VF | PCI function/resource 隔离 | confidential memory/firmware |
| MIG | GPU partition | CC/attestation |
| vGPU | 厂商 VM 产品 | CC 依产品 |
| API remoting | API mediation | Host 不见敏感数据 |
| Host container + MIG | resource 隔离 | CPU/GPU confidential |

## 12. Security Review Matrix

### 12.1 Trust

```text
CPU TEE:
VMM:
GPU firmware:
vGPU manager:
renderer/driver:
network/collective:
```

### 12.2 Memory

```text
guest private:
shared ring/MMIO:
VRAM/HBM:
bounce/SWIOTLB:
context save:
checkpoint:
```

### 12.3 Identity

```text
device attestation:
firmware measurement:
PASID/RID:
IOMMU domain:
remote peer:
```

### 12.4 Output

```text
display capture:
remote encoder:
telemetry:
crash dump:
model cache:
```

## 13. 常见错误结论

### “VM 启用 SEV/TDX，所以 GPU workload 机密”

错。CPU RAM 的边界不等于 GPU/HBM/Host renderer。

### “VFIO 直通 GPU，所以 Host 看不见”

错。需 GPU private memory/command/firmware attestation。

### “有 IOMMU 就能防止 Host 读取”

错。IOMMU 限制 DMA 地址，不提供数据加密和对 Host CPU 的机密性。

### “MIG/VF 是 confidential”

错。资源 partition 与 confidential trust domain 是不同维度。

### “Socket 加密让 renderer 安全”

错。renderer 在端点解密后仍可读 payload。

### “attestation report 存在就完成”

错。必须验证 freshness、policy、firmware、设备配置和 revocation。

## 14. 验证清单

1. 模型/权重在哪些页是 private？
2. Host 是否能读 HBM/VRAM？
3. Host renderer/driver 是否解析 command/buffer？
4. DMA 是否使用 bounce/SWIOTLB 明文？
5. MMIO/ring/fence 是否泄漏敏感 metadata？
6. GPU attestation 覆盖哪些 firmware/config？
7. CPU+GPU 是否组合 attestation？
8. key release 是否在 attestation 后？
9. checkpoint/migration 是否加密？
10. GPU/CPU TLB/IOMMU invalidate 顺序？
11. collective/peer 是否在同一 trust domain？
12. telemetry/crash/display/VDI 如何保护？
13. 产品 support matrix/version 是什么？
14. 安全声明是否有独立测试/评审证据？

## 15. 检查题

1. CPU private/shared memory 的边界是什么？
2. 为什么 device DMA 需要 SWIOTLB，可能带来什么风险？
3. API remoting 为什么通常破坏 Host opaque GPU 假设？
4. VFIO、SR-IOV、MIG 与 confidential GPU 分别在哪一层？
5. private HBM/context save 需要哪些保护？
6. CPU attestation 为什么不能单独证明 GPU 可信？
7. GPU attestation nonce/freshness 为什么重要？
8. collective 中间 tensor 如何暴露模型？
9. checkpoint migration 为什么要重新 attest？
10. 如何判断 Host 回读/remote encode 是合法输出还是泄漏？

## 16. 与其他专题

- 安全威胁模型：  
  [GPU 虚拟化隔离、安全与故障域](gpu-virtualization-security.md)
- 显存与 DMA：  
  [GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)
- PCIe/IOMMU：  
  [PCIe、IOMMU、ATS/PASID 与 GPU P2P](pcie-iommu-ats-p2p.md)
- 平台与 VFIO：  
  [GPU 虚拟化平台比较](../virtio-gpu/platform-comparison.md)
- 生命周期：  
  [GPU VM 生命周期、热插拔、Checkpoint 与迁移](vm-lifecycle-migration.md)
- AI 安全：  
  [AI GPU 计算栈与虚拟化边界](ai-compute-virtualization.md)
