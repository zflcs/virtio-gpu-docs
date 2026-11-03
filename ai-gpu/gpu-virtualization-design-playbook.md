# GPU 虚拟化架构选型与评审 Playbook

## 1. 用途

本文不是另一种技术机制介绍，而是把已有源码和专题转成可执行的架构流程：

```text
workload and business constraints
  -> required software contract
  -> isolation and topology requirements
  -> lifecycle requirements
  -> candidate GPU sharing mechanisms
  -> evidence gates
  -> reference architecture
  -> rollout and rollback
```

目标不是寻找一个“始终最好”的虚拟化方案，而是找出在指定 workload、
硬件和运营约束下可证明可接受的方案。

本文不执行实验。任何性能、隔离、迁移或支持矩阵结论都必须来自对应版本的
源码、官方文档、发行说明和实测证据。

## 2. 先明确产品目标

### 2.1 Workload

- 图形、VDI、Android、CI；
- Vulkan compute；
- CUDA/HIP 训练；
- 微调；
- prefill/decode 推理；
- batch/offline；
- scientific/HPC；
- 多卡 collective。

### 2.2 Runtime

- 现有 CUDA/HIP 二进制；
- 可重新编译的 Vulkan；
- 自研 UMD/KMD；
- 厂商私有库；
- 容器镜像；
- Windows/Linux Guest。

### 2.3 资源形态

- 一 VM/容器一整卡；
- 多 VM 共享一卡；
- 单卡多实例；
- 多卡同租户；
- 多租户多卡 collective；
- 异构 GPU 池。

### 2.4 业务目标

- 最大兼容性；
- 最大吞吐；
- 最低 tail latency；
- 最高租户密度；
- 强安全隔离；
- 可迁移/可恢复；
- 跨厂商；
- 运维简单；
- 成本/许可可接受。

这些目标经常冲突，必须明确优先级。

## 3. 六层架构视图

### 3.1 设备暴露层

- virtio-gpu；
- 物理 PCI GPU；
- SR-IOV VF；
- mdev/vGPU；
- MIG instance；
- container device。

### 3.2 软件契约层

- CUDA driver/runtime；
- HIP/ROCr/KFD；
- Vulkan/OpenGL；
- custom driver UAPI；
- container Host driver。

### 3.3 命令/队列层

- API command stream；
- native UAPI；
- virtio control queue；
- physical queue/doorbell；
- Host proxy queue。

### 3.4 内存层

- Guest RAM/GPA；
- Host memory；
- IOMMU；
- GPU VA/PTE；
- VRAM/HBM/GTT；
- pin/migration。

### 3.5 调度/隔离层

- engine time slice；
- hardware context；
- VF；
- MIG；
- vGPU scheduler；
- MPS/time slicing。

### 3.6 生命周期/运营层

- hotplug；
- reset/error；
- checkpoint；
- migration；
- telemetry；
- upgrade；
- quota/billing；
- security.

任何候选方案都要在这六层上填完整答案。

## 4. 候选架构原型

### 4.1 VFIO 整卡直通

```text
VM
  -> Guest native GPU driver
  -> physical GPU
  -> Host IOMMU
```

适合：

- 最大 CUDA/HIP 兼容性；
- 单租训练；
- 对隔离要求高；
- 可接受一 VM 独占。

主要风险：

- 密度低；
- reset/IOMMU group/ACS；
- migration 依设备；
- Host 可能失去 GPU。

### 4.2 SR-IOV VF

```text
PF manager
  -> VF
  -> VFIO/platform assignment
  -> Guest driver
```

适合：

- 需要多 VM；
- 硬件和 driver 支持 VF；
- profile/HBM/engine 切分满足要求。

主要风险：

- VF 支持矩阵；
- 共享 firmware/engine/fabric；
- P2P/ACS；
- migration 依 PF/platform。

### 4.3 Vendor vGPU

```text
Host GPU manager
  -> virtual GPU device
  -> Guest vGPU driver
  -> Host scheduler
```

适合：

- 需要成熟 VM 管理/图形/AI 产品栈；
- 接受厂商许可和支持矩阵；
- 需要 profile、远程显示和管理工具。

主要风险：

- 版本绑定；
- licence；
- 特定 GPU/hypervisor；
- migration 能力因产品而异。

### 4.4 MIG

```text
Physical GPU
  -> GPU instance/compute instance partitions
  -> VM assignment or container device
```

适合：

- 共享数据中心 GPU；
- 需要 compute/HBM 空间分区；
- 目标 profile 满足 workload。

主要风险：

- media/P2P/profile 限制；
- VM assignment 还需额外机制；
- 共享 PCIe/firmware/reset 域；
- 重配需排空。

### 4.5 MPS/time slicing

```text
Host kernel driver
  -> multiple Host processes/contexts
  -> shared GPU
```

适合：

- 受控容器；
- 小模型推理；
- 开发/CI；
- 能接受弱 fault/security 隔离。

主要风险：

- HBM/queue/fault 共享；
- tail latency；
- side channel；
- context 清理和 OOM。

### 4.6 Virtio-GPU 图形/API remoting

```text
VM
  -> Guest UMD
  -> virtio-gpu
  -> QEMU/renderer
  -> Host graphics API/native driver
```

适合：

- Linux 图形；
- VDI/CI；
- Android/Cuttlefish；
- Vulkan/OpenGL API remoting；
- 教学和源码研究。

主要风险：

- CUDA/HIP 兼容有限；
- renderer parser 安全；
- submit/fence overhead；
- migration blocker、WSI 和版本矩阵。

### 4.7 Virtio-GPU native DRM/ROCm context

适合：

- 需要在 VM 内保留更多 native driver UAPI；
- 能接受 Host native driver 版本耦合；
- 目标是研究/受控部署。

主要风险：

- Guest/Host UAPI 强耦合；
- 资源/fence/reset；
- ROCm 全栈未稳定。

## 5. 决策 Gate

### Gate 0：威胁模型

先回答：

- Guest kernel 是否可信？
- 同租户/跨租户是否互不信任？
- Host 是否必须看不到模型/用户数据？
- 是否允许 Host renderer 解析 command？
- 是否允许 Host GPU driver/firmware 共享？

如果要求 Host 看不到明文数据，普通 API remoting 和共享 Host driver 往往不
满足；需要具体 confidential GPU 方案并验证整个数据路径。

### Gate 1：软件契约

枚举：

- 现有二进制；
- runtime/library；
- driver UAPI；
- kernel/ISA；
- extension/private API。

若应用依赖完整 CUDA/HIP/NCCL，优先保留 native device，而不是假设 Vulkan
API remoting 可替代。

### Gate 2：内存和 engine

确定：

- HBM 容量和 bandwidth；
- pinned/system memory；
- queue/engine；
- SVM/userptr；
- P2P；
- migration/eviction；
- fault/retry；
- preemption。

### Gate 3：隔离

分别定义：

- memory；
- execution；
- control plane；
- fault/reset；
- data residue；
- timing；
- management。

“有 MIG”或“有 VF”不能作为完整威胁模型。

### Gate 4：topology

- GPU-NIC affinity；
- NVLink/PCIe/XGMI；
- NUMA；
- MIG/VF peer；
- IOMMU/ACS；
- collective transport。

单 GPU 测试不能证明分布式 AI 能力。

### Gate 5：生命周期

- hotplug；
- reset；
- suspend/resume；
- checkpoint；
- snapshot；
- live migration；
- driver upgrade；
- rollback。

区分“可以重启恢复”和“可以 live migrate”：

```text
restart/reset
  ≠ suspend/resume
  ≠ snapshot
  ≠ live migration
```

### Gate 6：性能

至少验证：

- GPU-bound；
- launch-bound；
- HBM-bound；
- copy-bound；
- sync-bound；
- multi-tenant P99；
- collective。

性能测试必须先通过正确性和路径验证。

### Gate 7：运营与成本

- Host/Hypervisor support；
- driver/firmware 升级；
- GPU inventory；
- MIG/VF/vGPU profile；
- licence；
- quota/billing；
- telemetry；
-故障处置；
- capacity headroom；
- 多版本共存。

## 6. 选择矩阵

| 需求 | 首选 | 次选 | 避免误判 |
| --- | --- | --- | --- |
| 现有 CUDA/PyTorch 二进制 | VFIO/VF/vGPU | 受验证容器 Host driver | 不要假设 Vulkan/Venus 兼容 |
| 现有 HIP/ROCm | VFIO/VF/vGPU | 成熟 ROCm 虚拟化栈 | ROCm proposal 未稳定 |
| Linux 图形/VDI | virtio-gpu/vGPU | VF | API remoting 仅覆盖图形 |
| Android/Cuttlefish | GFXStream + Rutabaga | Venus | 不要外推 CUDA |
| 多 VM 密度 | SR-IOV/vGPU/MIG | API remoting/受控共享 | 分配层不等于隔离 |
| 多容器密度 | MIG/VF/MPS/time slice | 独立 GPU | MPS/time slice 非强 VM |
| 单租最高兼容 | VFIO/整卡 | vGPU/VF | 密度和迁移代价 |
| 强安全多租 | VF/VFIO 隔离/vGPU/MIG | 独立设备/受控 renderer | IOMMU 不是全部安全 |
| 低延迟 decode | 性能优先且少 round-trip | 分区/独占 | 大 GEMM 结果不能代表 decode |
| 多卡训练 | native GPU/VF/vGPU | MIG 支持组合 | 单卡 benchmark 不足 |
| 迁移 | 明确支持 migration 的产品 | 应用 checkpoint | snapshot API 不等于 live migration |

## 7. 工作负载蓝图

### 7.1 Linux 图形桌面/VDI

显示、Compositor、dma-buf WSI 和远端编码的独立验收面见
[GPU 显示、Compositor、WSI 与 VDI](display-wsi-virtualization.md)。

```text
VM
  -> Guest Mesa/Vulkan/OpenGL
  -> virtio-gpu
  -> QEMU + renderer
  -> Host GPU
```

验收：

- 2D/3D 正确；
- scanout/cursor；
- compositor/WSI；
- suspend/reset；
- 多分辨率/多显示器；
- GPU fault；
- 迁移是否声明支持。

### 7.2 Android/Cuttlefish

```text
Guest Android graphics
  -> GFXStream/Venus
  -> Rutabaga/renderer
  -> Host GPU
  -> cross-domain/WSI/display
```

验收：

- GLES/Vulkan；
- dma-buf/cross-domain；
- compositor；
- native WSI；
- host Xid/reset；
- frame pacing。

### 7.3 单租 LLM serving

候选：

- 独立 GPU/整卡 VFIO；
- vGPU；
- VM/VF；
- 容器 Host driver。

验收：

- 模型加载；
- prefill/decode；
- first-token 和 P99；
- KV cache；
- graph/capture；
- model upgrade；
- OOM/device lost。

### 7.4 高密度推理

候选：

- MIG；
- vGPU；
- VF；
- MPS/time slicing；
- 多模型服务进程。

验收：

- HBM headroom；
- decode P99；
- prefix/KV locality；
- 多租户干扰；
- context cleanup；
- pod/VM termination；
- fault 隔离。

### 7.5 多租训练

候选：

- VFIO/VF/vGPU；
- MIG 且支持所需 collective；
- 明确 profile 的容器。

验收：

- topology/P2P；
- NCCL/RCCL；
- step time；
- checkpoint；
- slow rank；
- tenant interference；
- reset recovery。

### 7.6 CI/开发

优先：

- 可重建环境；
- 可观测 API remoting；
- 低密度要求；
- 明确哪些测试允许软件渲染。

验收：

- API/ICD 选择；
- 像素/数值正确性；
- 与生产 backend 差异；
- 资源清理；
- 多 job 隔离。

## 8. 评分框架

### 8.1 先做 hard gate

下列项不满足时不应进入评分：

- required API/ABI；
- supported GPU/profile；
- HBM quota；
- collective/P2P；
- migration；
- security/compliance；
- vendor support。

### 8.2 加权评分

示例权重：

| 维度 | 示例权重 |
| --- | --- |
| 正确性/兼容性 | 30 |
| 安全/隔离 | 20 |
| 性能/P99 | 20 |
| 密度/成本 | 10 |
| 生命周期/迁移 | 10 |
| 运维/支持 | 10 |

权重应来自业务需求，不应为了让某方案胜出而修改。

### 8.3 证据等级

| 等级 | 证据 |
| --- | --- |
| E0 | 博客/宣传 |
| E1 | 官方架构文档 |
| E2 | 固定版本源码/UAPI |
| E3 | 全栈 ABI 对齐、维护者接受 |
| E4 | 发行版/产品可用和 support |
| E5 | 目标硬件完整验收数据 |

功能、性能和安全性分别评级，不要求一个等级覆盖所有维度。

## 9. 架构评审产出模板

### 9.1 Context

```text
Workload:
Model/frame/API/batch:
Tenant model:
Security/compliance:
Performance SLO:
Lifecycle requirements:
Hardware/software versions:
```

### 9.2 Architecture

```text
Device exposure:
Guest software:
Host software:
Command path:
Memory path:
DMA/IOMMU:
GPU VA/cache:
Queue/scheduler:
Completion/fence:
Display:
```

### 9.3 Isolation and failure

```text
Hardware partition:
Software mediation:
Data residue:
Tenant DoS:
Fault domain:
Reset/recovery:
Migration/snapshot:
Telemetry security:
```

### 9.4 Decision

```text
Chosen option:
Rejected alternatives and why:
Hard gates:
Known unsupported features:
Evidence level:
Measurement plan:
Rollout/rollback:
Owner:
Expiry/re-review date:
```

## 10. 分阶段落地

### 10.1 文档和原型

- API/ABI inventory；
- device/topology inventory；
- security review；
- no-fallback correctness；
- failure matrix。

### 10.2 单租 canary

- 一个 workload；
- P50/P99；
- HBM/copy；
- reset；
- telemetry；
- rollback to native。

### 10.3 多租 shadow

- 第二个租户；
- shared engine/HBM；
- P99 interference；
- fault isolation；
- quota。

### 10.4 生产扩展

- autoscaling；
- upgrade；
- capacity headroom；
- license；
- incident response；
- DR/checkpoint。

## 11. 反模式

### “先买 GPU，再决定虚拟化”

应从 workload、API、隔离和 lifecycle 反推平台。

### “用 GPU-bound benchmark 选所有场景”

会掩盖 launch/fence/memory/topology 问题。

### “MIG/vGPU/SR-IOV 都是分区”

它们分别属于硬件 partition、VM 产品栈、PCIe function，层次不同。

### “DRA/device plugin 已是隔离”

它们主要解决分配/调度。

### “Host renderer 代理 Vulkan 就能跑所有 AI”

CUDA/HIP 库、graph、IPC/P2P、NCCL 需要额外 ABI。

### “能 snapshot 就能 live migrate”

设备状态、Host object、外部资源和双 owner 都需要明确。

### “安全由 IOMMU 保证”

DMA 隔离之外还有 parser、firmware、VRAM、reset、side channel 和数据清理。

### “单 VM 通过就说明多租生产就绪”

还需要 P99、fault、resource exhaustion、upgrade 和 rollback。

## 12. 专家验收

能够做到以下四点，才具备独立架构判断力：

1. 从应用/API 追到 kernel driver、queue、GPU engine 和 memory；
2. 根据 workload 选择 VFIO/SR-IOV/vGPU/MIG/MPS/virtio-gpu，并解释取舍；
3. 设计正确性、性能、隔离、fault、migration 证据矩阵；
4. 识别方案文档中的 capability overclaim、ABI mismatch 和错误隔离假设。

## 13. 与其他专题的关系

- 四条路径源码：  
  [四路径源码横向对照](../virtio-gpu/source-trace-cross-path.md)
- 平台机制比较：  
  [GPU 虚拟化平台比较](../virtio-gpu/platform-comparison.md)
- 显存与 coherence：  
  [GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)
- 执行与调度：  
  [GPU 执行模型、队列、调度与故障隔离](gpu-execution-model.md)
- 安全：  
  [GPU 虚拟化的隔离、安全与故障域](gpu-virtualization-security.md)
- 生命周期和迁移：  
  [GPU VM 生命周期、热插拔、Checkpoint 与迁移](vm-lifecycle-migration.md)
- 性能：  
  [GPU 虚拟化性能分析与可观测性](gpu-performance-observability.md)
- 云原生：  
  [云原生 AI 中的 GPU 共享与虚拟化](cloud-native-gpu-sharing.md)
- API/ABI：  
  [AI Runtime、API 与虚拟化兼容性](ai-runtime-api-compatibility.md)
