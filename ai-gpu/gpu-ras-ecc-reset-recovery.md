# GPU RAS、ECC、Reset 与 AI Job 恢复

## 1. 为什么可靠性是虚拟化问题

多租户 GPU 中，同一个物理故障可能表现为：

- 一个 shader/kernel 结果错误；
- 一个 queue timeout；
- 一个 process device lost；
- 一个 VF/MIG instance 不可用；
- 整个 GPU reset；
- sibling tenant 一起失败；
- PCIe AER/FLR；
- Host reboot 或 VM 崩溃。

专家需要先定义 fault domain，再定义 recovery unit。

本文不执行 ECC、RAS、reset 或故障注入实验。

## 2. 故障分类

| 类别 | 典型含义 | 可能恢复粒度 |
| --- | --- | --- |
| Corrected ECC | 错误已纠正，可能有性能/健康影响 | allocation/page/telemetry |
| Uncorrected ECC | 数据不可信 | page/context/device |
| Poison | 读到已知坏数据 | page/process |
| GPU page fault | VA/PTE 缺失或权限错 | faulting work/context |
| IOMMU/PRI fault | device DMA translation fault | IOMMU mapping/context |
| Queue hang | queue 不推进/超时 | queue/engine |
| Firmware hang | scheduler/firmware 失效 | engine/GPU |
| PCIe AER | link/protocol error | function/hierarchy |
| Thermal/power | 降频或保护停机 | workload/host policy |
| NIC/fabric fault | collective transport 失败 | channel/communicator |
| Xid/device lost | NVIDIA 设备错误事件 | context/device |

“corrected”不代表零成本；持续 ECC 可能需要 page retirement 或 RMA。

## 3. Corrected vs Uncorrected

### 3.1 Corrected

硬件修复了读取，但：

- 影响 bandwidth；
- 表明硬件劣化；
- 应监控 rate/trend；
- 超过阈值可能需要 page retirement；
- 多租户需要 health isolation。

### 3.2 Uncorrected

数据不可信。应用不能简单重试同一 allocation：

- 需要隔离 poison page；
- 使访问该 page 的 workload 失败；
- Guest 需要明确 memory exception；
- checkpoint 若已包含坏数据也不可恢复。

## 4. AMDGPU RAS

### 4.1 Public interfaces

[AMDGPU RAS 文档](../../linux/Documentation/gpu/amdgpu/ras.rst:1) 说明：

- sysfs informational queries；
- debugfs error injection；
- RAS control interface；
- unrecoverable error reboot behavior；
- per-IP error counts；
- EEPROM table；
- VRAM bad pages interface。

### 4.2 Per-IP health

RAS 不是全局一个 bool：

- GFX；
- MMHUB；
- SDMA；
- VCN/JPEG；
- IH；
- memory/fabric blocks。

不同 IP 的 fault 影响不同 queue/engine。

### 4.3 Bad pages

VRAM bad-page tracking 用来：

- 标记 retired pages；
- 避免重新分配；
- 传递到 host-managed VF recovery；
- 支持 attestation/telemetry（产品支持时）。

Guest 收到 bad page 不应继续把 page 当有效 data。

### 4.4 RAS in VF

AMD MxGPU VF 可通过 PF/VF feature 进行：

- RAS capability negotiation；
- telemetry；
- bad page；
- CPER/poison；
- PTL。

支持集合由 Host/GIM/platform feature flags 决定。

详见 [AMD GPU SR-IOV 与 MxGPU](amd-gpu-sriov-mxgpu.md)。

## 5. KFD memory/hardware exception

### 5.1 Error types

[`kfd_ioctl.h`](../../linux/include/uapi/linux/kfd_ioctl.h:283) 定义：

- `KFD_MEM_ERR_NO_RAS`；
- `KFD_MEM_ERR_SRAM_ECC`；
- `KFD_MEM_ERR_POISON_CONSUMED`；
- `KFD_MEM_ERR_GPU_HANG`。

Memory exception data 可带：

- NotPresent；
- ReadOnly；
- NoExecute；
- imprecise；
- fault VA/GPU ID。

### 5.2 Reset event

[`kfd_signal_reset_event()`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_events.c:1159)：

- reset cause 可为 ECC 或 GPU hang；
- whole GPU reset 设置 `memory_lost=1`；
- 向使用该 device 的 KFD processes signal HW exception；
- ECC reset 还 signal memory exception；
- 可按 per-process queue reset 状态筛选。

### 5.3 Poison consumption

Poison-consumed event 可以：

- 默认向进程发 SIGBUS；
- 对 opt-in 应用先给 RAS event；
- 设置 safety timeout，避免进程无限忽略；
- `memory_lost=1`。

这说明“收到 event”与“VRAM 内容仍有效”是两个问题。

## 6. GPU page fault decoding

[AMDGPU debugging 文档](../../linux/Documentation/gpu/amdgpu/debugging.rst:27)
给出：

- gfxhub/mmhub；
- vmid/PASID；
- process/pid；
- GPU VA；
- client ID；
- permission bits；
- read/write；
- mapping/walker error。

Fault decoding 应回答：

1. 哪个 hub/engine；
2. 哪个 process/VM；
3. VA 是否映射；
4. 权限/Binding 是否错；
5. descriptor/shader 是否生成非法 VA；
6. 是否 Host renderer 的第二层 VM。

VM/fault 不一定代表硬件损坏。

## 7. Intel Xe reliability evidence

### 7.1 Device wedged/reset

[`xe_device_types.h`](../../linux/drivers/gpu/drm/xe/xe_device_types.h:54)
描述 critical error / GT reset failure / firmware failure 可导致 device
wedged mode。

包含：

- reset in progress；
- wedged flag；
- inconsistent reset policy state between GTs。

### 7.2 SIGID fault classes

[`xe_sigid_abi.h`](../../linux/drivers/gpu/drm/xe/abi/xe_sigid_abi.h:130)
把 operator-relevant fault site 分类：

- GT TDR/engine hang；
- VM bind/page/GTT fault；
- PCIe/IOMMU/MMIO fault；
- PCIe interface；
- device memory；
- compute/shader core；
- fabric；
- SoC internal。

这是比“device lost”更细的故障归因。

### 7.3 GT stats

[`xe_gt_stats_types.h`](../../linux/drivers/gpu/drm/xe/xe_gt_stats_types.h:13)
记录：

- chained/parallel/valid page faults；
- SVM page fault/TLB invalidate；
- page fault 处理时间；
- migration/copy bytes/time；
- prefetch fault；
- LR queue suspend/wait。

可用于关联性能下降和 fault storm。

## 8. PCIe AER/FLR

[`pcieaer-howto.rst`](../../linux/Documentation/PCI/pcieaer-howto.rst:123)：

- error recovery 需要 driver callbacks；
- correctable/uncorrectable 有不同策略；
- fatal error 可能需要 endpoint/upstream port/hierarchy reset；
- 非 fatal 可由 driver 决定是否 reset。

Virtualization:

- Guest 可能只见 vendor/virtual device，不见 Host AER；
- VFIO 可能把部分错误通知 Guest；
- VF/VGPU 的 reset 可能由 PF/manager 发起；
- Guest FLR/VF reset 不等于 Host physical reset。

## 9. Reset 层级

### 9.1 Work retry

- kernel/launch failure；
- fence timeout；
- context 仍有效；
- 重放工作，避免重复副作用。

### 9.2 Context reset

- GPU fault、poison page、descriptor error；
- 重建 context/stream/queue；
- tensor allocation 可能仍有效。

### 9.3 Queue/engine reset

- queue hang；
- preemption failure；
- isolated queue/engine；
- sibling queue 影响依硬件。

### 9.4 VF reset/FLR

- VF queue/VRAM/context；
- Host PF reinit；
- P2P/XGMI topology update；
- Guest device lost/reprobe。

### 9.5 Whole GPU reset

- firmware/scheduler failure；
- uncorrectable memory/fabric fault；
- all contexts；
- VRAM loss；
- sibling tenant impact。

### 9.6 Host/VM restart

- GPU device lost；
- driver/VM reboot；
- workload checkpoint restore。

## 10. AI job recovery

### 10.1 Recovery hierarchy

```text
retry kernel
  -> rebuild stream/context
  -> recreate communicator
  -> restart training step
  -> restore process/VM
  -> reschedule job
```

更高层恢复成本更高，且可能丢失更多状态。

### 10.2 Training

Checkpoint 需要：

- weights；
- optimizer；
- gradient；
- RNG；
- data loader；
- step/scheduler；
- collective state；
- allocator/workspace reconstruction。

如果 ECC poison 在 checkpoint 数据中，校验和重要。

### 10.3 Inference

Decode KV cache 可能是不可恢复状态：

- 迁移回 CPU；
- 重新 prefill；
- 路由到 replica；
- 返回错误。

需要定义 request-level idempotency。

### 10.4 Distributed

一个 rank reset 后：

- NCCL/RCCL communicator 可能失效；
- collective 必须 abort/recreate；
- other ranks 不应无限等待；
- topology 可变化；
- timeout 与 error propagation 必须一致。

## 11. Health telemetry

### 11.1 Correctness/latency

- ECC corrected rate；
- uncorrected/poison；
- page retired；
- fault storm；
- xid/SIGID/AER；
- reset count/cause；
- memory lost。

### 11.2 Isolation

- per GPU；
- per partition/VF；
- per process/context；
- per queue。

可用性告警不能只给整卡平均。

## 12. Failure scenarios

| 场景 | 错误做法 | 正确方向 |
| --- | --- | --- |
| corrected ECC spike | 忽略 | 监控趋势/retire |
| poison page | 重试同一 page | isolate/signal |
| GPU page fault | 立即整卡 reset | inspect PTE/permissions |
| queue timeout | 永远等 | preempt/reset queue |
| one rank lost | all ranks hang | abort collective |
| whole GPU reset | resume same stream | device lost/reprobe |
| VF reset | assume whole GPU lost | PF/VF scope |
| FLR VRAM lost | restore stale pointers | reinit/checkpoint |

## 13. Verification checklist

1. fault domain 是 queue/context/VF/MIG/GPU?
2. corrected 还是 uncorrected?
3. data 是否 poison?
4. reset cause 和 memory_lost?
5. page retirement/telemetry?
6. Guest/Host error 如何传播?
7. pending DMA/TLB/IRQ/fence 如何清理?
8. context/queue/VRAM 哪些重建?
9. collective 如何出错?
10. checkpoint 有效性?
11. 其他 tenant 是否受影响?
12. 设备何时可重分配?

## 14. 常见错误

### “ECC corrected，所以完全正常”

错。持续 error 影响性能和硬件健康。

### “page fault 就是硬件坏”

错。先解码 VA/PTE/permission。

### “queue reset 等于 GPU reset”

错。

### “Guest FLR 等于 Host physical reset”

错。

### “全部 reset 后重新提交同一 stream”

错。context/fence/device generation 可能已失效。

### “一个 rank 失败后其他 rank 继续等”

错。collective 必须 abort/recreate。

## 15. 与其他专题

- 诊断：  
  [GPU 虚拟化故障诊断](gpu-virtualization-diagnostics.md)
- 生命周期：  
  [GPU VM 生命周期与迁移](vm-lifecycle-migration.md)
- KFD：  
  [AMD KFD/ROCr](amdkfd-source-trace.md)
- AMD VF：  
  [AMD GPU SR-IOV 与 MxGPU](amd-gpu-sriov-mxgpu.md)
- Intel VF：  
  [Intel Xe GPU 虚拟化与 SR-IOV](intel-xe-sriov.md)
- 安全：  
  [GPU 虚拟化安全](gpu-virtualization-security.md)
- NCCL/RCCL：  
  [Collective 源码](nccl-rccl-source-trace.md)
