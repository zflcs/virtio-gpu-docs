# 云原生 AI 中的 GPU 共享与虚拟化

## 1. 文档定位

本文把前面的 GPU 虚拟化知识映射到容器和 Kubernetes，重点区分：

- VM 的设备虚拟化；
- 容器的 Host 进程共享；
- Kubernetes 资源分配；
- GPU vendor 的 MIG/MPS/time slicing/vGPU；
- 真正提供内存、执行和故障隔离的层。

结论先行：

```text
Kubernetes allocation label
  ≠ device virtualization
  ≠ hardware partition
  ≠ memory isolation
  ≠ fault isolation
  ≠ security isolation
```

本文不运行集群、容器、GPU workload 或 benchmark。Kubernetes vendor API 和
支持矩阵会变化，具体实现必须核对对应版本官方文档。

## 2. VM、容器和进程的边界

| 形态 | Guest/Host kernel | 设备暴露 | 主要隔离边界 | 典型 GPU 方式 |
| --- | --- | --- | --- | --- |
| VM | 独立 Guest kernel | virtio、VF、mdev、vGPU | Hypervisor + IOMMU | VFIO、SR-IOV、vGPU、virtio-gpu |
| 容器 | 共享 Host kernel | `/dev/dri`、`/dev/kfd` 等 | namespace、cgroup、LSM、device permission | exclusive device、MIG、MPS、time slicing |
| Host process | 共享 Host kernel | 直接打开 driver fd | OS 进程权限 | 同容器机制，但无编排层 |

容器没有独立 Guest kernel，所以：

- Guest 内 GPU driver 漏洞模型不存在；
- Host GPU driver 和 kernel 是共享攻击面；
- 一个 container 能访问的 device node 可能拥有很高权限；
- device cgroup/权限只控制“能否打开设备”，不自动限制 GPU 内部 engine；
- GPU memory/queue/fault 隔离仍由 vendor driver/hardware 提供。

## 3. 分配层不是隔离层

### 3.1 分配层负责什么

- 发现 GPU；
- 把资源交给 Pod/container；
- 设置环境变量/UUID/device path；
- 处理调度、claim、attach；
- 暴露健康/拓扑信息；
- 标记资源是否可分配。

### 3.2 隔离层负责什么

- GPU page table/VM；
- HBM/VRAM 配额；
- queue/context/engine 隔离；
- fault/reset 范围；
- cache/LDS/GPR 数据残留；
- time/bandwidth fairness；
- PCIe/IOMMU/P2P 边界。

一个资源被成功分配给 Pod，只证明调度器认为它可用，不证明上述隔离属性。

## 4. 常见 GPU 共享机制

### 4.1 整设备独占容器

容器获得一个完整 GPU device 和必要的 render/compute node。

特点：

- 兼容性最好；
- 性能路径最接近 Host native；
- 一个 workload 独占整卡；
- 容器间不共享同一 GPU context 调度；
- 仍需处理 device node 权限、driver 版本和 GPU fault。

如果 Host kernel driver 存在漏洞，整卡独占也不能提供 VM 级隔离。

### 4.2 MIG

MIG 把支持的数据中心 GPU 划分为 GPU instance/compute instance。

容器场景常见分配：

- 一个 MIG instance；
- 一个 MIG compute instance；
- 通过 vendor device UUID 指定。

必须核对：

- 计算/SM、HBM、L2 slice；
- media engine 能力；
- P2P/NVLink；
- fault/reset 范围；
- 同一物理 GPU 上其他 MIG workload 的干扰；
- MIG 配置变更是否需要排空全部 workload。

MIG 是硬件 partition，但 Kubernetes 资源名本身不是隔离证明。

### 4.3 MPS

MPS 面向 Host 上多个进程共享一个 GPU：

- 减少 context switch；
- 允许部分并发；
- 可设置 active thread percentage 等策略；
- 仍共享 GPU 内存/system state/fault domain；
- 通常不适合不可信多租户的强安全边界。

MPS 对 decode、小 kernel 并发可能有帮助，也可能与 memory capacity 冲突。

### 4.4 Time slicing

time slicing 通过时间片轮转共享 GPU：

- 便于简单提高分配数量；
- 每个 workload 不是持续获得完整的 GPU；
- tail latency 和 fairness 取决于 scheduler；
- memory、fault、cache 和 security 仍共享；
- 多个 Pod 看到同一物理 GPU 不等于独立 device。

### 4.5 SR-IOV VF

VF 是 PCIe function，通常更适合 VM 或受控容器：

```text
PF -> VF -> VFIO/device plugin -> Guest/container driver
```

需要核验：

- VF profile 和 HBM quota；
- engine 调度；
- P2P/ACS/IOMMU；
- reset/error 范围；
- VF 生命周期和迁移。

将 VF 交给容器时，容器仍使用 Host kernel driver；只有在 VM 中才拥有独立
Guest driver。

### 4.6 Vendor vGPU

vGPU 通常面向 Hypervisor VM，包括 Host manager、Guest driver、profile、
调度和产品支持矩阵。

某些平台可能把它与容器/VM 管理层组合，但：

- 不能假设 container 中可用；
- 不能假设它与 MIG/SR-IOV 同层；
- 必须核对 licence、guest OS、hypervisor 和 profile。

### 4.7 Virtio-GPU/API remoting

在云原生中，virtio-gpu 更适合：

- VM 内图形/CI；
- GUI/VDI；
- Android/模拟器；
- API remoting 研究；
- 特定受控 compute runtime。

容器通常直接使用 Host `/dev/dri`/vendor device，不需要 virtio-gpu 作为中间
设备；若把 virtio-gpu 暴露给 microVM，则又回到 VM 生命周期。

## 5. Kubernetes 资源层的角色

### 5.1 Device Plugin

典型概念：

- node 上 agent 发现 GPU/MIG；
- 向 kubelet 宣告可分配资源；
- kubelet 将 device 分配给 Pod；
- plugin 负责 device node、env 和可用性。

Device plugin 主要解决发现和分配，不定义 GPU context/fault 隔离。

### 5.2 Dynamic Resource Allocation（DRA）

DRA 的资源 claim/分配模型适合表达：

- device class；
- 属性/能力；
- 多个设备组合；
- topology/affinity；
- claim 生命周期；
- vendor structured parameters。

具体 API 以所用 Kubernetes 版本为准。DRA 的调度成功仍不等于 MIG/VF 等
底层资源已具备安全隔离。

### 5.3 CDI

CDI 用设备规格描述需要注入容器的：

- `/dev` nodes；
- mount；
- env；
- hook。

它是设备描述/注入机制，不是 GPU scheduler 或硬件分区器。

### 5.4 Extended resource 与 label

常见错误：

```text
nvidia.com/gpu: 1
gpu-profile: inference
sharing: mps
```

这些 label/resource 可能只影响调度。必须进一步确认：

- plugin 实际创建了什么 device view；
- MIG UUID、VF、MPS 或 time slice；
- container 获得哪些 device nodes；
- runtime/driver 是否识别正确；
- HBM/engine 是否真正受限。

## 6. GPU 生命周期与容器

### 6.1 Pod 启动

顺序通常涉及：

1. scheduler 选择 node；
2. device plugin/DRA 分配资源；
3. runtime 使用 CDI/device 配置启动 container；
4. driver 打开设备；
5. workload 创建 context/memory/queue；
6. 资源状态回写。

任何一步失败都可能导致：

- Pod Pending；
- device busy；
- UUID 不存在；
- driver/library 版本不匹配；
- container 可见 device 与 scheduler 记录不一致。

### 6.2 Pod 终止

终止时必须：

- 停止提交；
- 让框架完成/取消 collective；
- drain GPU work；
- 释放 CUDA/HIP/Vulkan context；
- 回收 MIG/VF/MPS slot；
- 清理 device visibility；
- 处理异常退出后残留 state。

强制 SIGKILL 可能让 kernel driver 延迟清理 GPU context。资源“已释放”应以
driver/device 实际回收为准，不只以 Pod phase 为准。

### 6.3 Driver 升级

Host kernel driver 升级通常需要：

- 排空该 GPU 上所有 workload；
- 关闭所有 context；
- 处理 persistence daemon；
- 更新 userspace library；
- 重新创建 MIG/VF/profile；
- 验证 CDI/plugin 资源重新通告。

滚动升级不能假设运行中的 context 会无损迁移。

## 7. Topology 与 NUMA

AI workload 的 locality 至少包括：

- GPU 与 CPU NUMA；
- GPU 与 Host memory；
- GPU 与 NIC；
- GPU 与 peer GPU（NVLink/PCIe/XGMI）；
- MIG instance 所在物理 GPU；
- VF 的 PCIe path；
- storage/cache locality。

Kubernetes 把两个 GPU 资源分配给同一 Pod，不代表它们：

- 属于同一 NVLink/XGMI hive；
- 支持 P2P；
- 与目标 NIC 最近；
- 属于允许多卡 collective 的同一 profile。

Linux NUMA policy 会影响系统内存分配和 locality；GPU/NIC topology 还要由
vendor discovery 和 NCCL/RCCL 的 topology 检查验证。

Linux 的 NUMA 节点和 locality 基础分别见
[`numa.rst`](../../linux/Documentation/mm/numa.rst:1) 和
[`numa_memory_policy.rst`](../../linux/Documentation/admin-guide/mm/numa_memory_policy.rst:1)。

## 8. Quota 与 fairness

### 8.1 需要分别限制

- GPU context 数；
- queue 数；
- HBM/VRAM bytes；
- pinned memory；
- copy engine bandwidth；
- compute time；
- interrupt/fence rate；
- page fault/migration rate；
- checkpoint/trace I/O；
- collective network bandwidth。

### 8.2 Cgroup 的边界

Linux
[`cgroup-v2`](../../linux/Documentation/admin-guide/cgroup-v2.rst:1) 可管理
CPU、memory 等资源；device/BPF 机制可控制设备访问。但“允许打开 GPU”与
“GPU 内部公平调度”是不同层。

GPU-specific cgroup 需要 vendor driver/hardware 支持。只限制 Host RAM 不能
阻止 Guest/container 用满 HBM。

### 8.3 Capacity 与 allocatable

`allocatable` 通常是分配口径，不一定是：

- 当前空闲 HBM；
- 可被 new context 使用的 capacity；
- 在 fault/migration/workspace 后的可用量；
- 多租户 P99 下的真实可分配量。

生产调度要区分 hard partition、soft quota 和 overcommit。

## 9. AI workload 设计

### 9.1 单租大模型训练

优先：

- 整 GPU；
- SR-IOV VF；
- vGPU；
- 足够 GPU 数与正确 collective fabric。

关键是完整厂商 runtime、P2P、NCCL/RCCL 和 checkpoint。

### 9.2 多租推理

候选：

- MIG；
- vGPU；
- VF；
- 受控 MPS/time slicing；
- 专用 serving process + API routing。

重点：

- decode P99；
- KV cache 容量；
- prefill/decode 资源竞争；
- prefix cache；
- fault 对其他租户的影响。

### 9.3 小模型/低利用率

可采用共享机制提高利用率，但要评估：

- context switching；
- memory capacity；
- latency SLA；
- failure isolation；
- 数据残余。

### 9.4 科学计算/批量离线

可接受更高 overcommit，但应明确：

- 抢占恢复；
- checkpoint 频率；
- 任务优先级；
- 结果确定性；
- 设备 reset 后重跑。

## 10. 安全边界

### 10.1 容器共享 Host kernel

高风险来源：

- privileged container；
- hostPath device/driver；
- `CAP_SYS_ADMIN`；
- 任意 GPU debug/profiler ioctl；
- 共享 `/dev/dri` 或 `/dev/kfd` 权限；
- device plugin 自身权限过大；
- 共享 host IPC/shared memory；
- Host driver/cmdline 配置误用。

### 10.2 多租户数据

需要保护：

- model weights；
- prompt/KV cache；
- embedding；
- checkpoint；
- telemetry；
- crash/core dump。

容器退出后仍需确认 GPU context/VRAM/cache 已清理，不能只删除 Pod 对象。

### 10.3 平台层

若平台提供 “GPU 资源池”：

- 控制面能创建/销毁 device；
- operator 能修改 node labels；
- device plugin 可访问设备；
- runtime hook 以 root 运行；
- 升级可重启 driver。

这些都是高权限信任边界，应与普通 workload 分开。

## 11. 可观测性

### 11.1 资源视图

记录：

- node GPU；
- MIG UUID/profile；
- VF identity；
- vGPU profile；
- MPS/time-slice slot；
- driver/library；
- CDI/device plugin 版本；
- container GPU assignment。

### 11.2 Workload 性能

记录：

- launch latency；
- kernel duration；
- HBM bandwidth；
- queue wait；
- context switching；
- page fault/migration；
- collective；
- GPU throttling/reset。

### 11.3 归因

“Pod 慢”可能来自：

- scheduler/CPU；
- HBM contention；
- MIG profile；
- MPS/time-slice interference；
- GPU fault；
- collective slow rank；
- WSI/display copy；
- vendor telemetry 定义不同。

要关联 Kubernetes resource identity 与 GPU workload ID，不能只看 Pod CPU。

## 12. 方案选择

| 场景 | 优先候选 | 主要原因 |
| --- | --- | --- |
| 单租训练、最大兼容 | 整 GPU / VF / vGPU | 完整 runtime 和多卡能力 |
| 多 VM AI | SR-IOV、vGPU、MIG+assignment | 更清晰设备/资源边界 |
| 多容器推理 | MIG、MPS、time slice、VF | 密度与隔离折中 |
| 高安全多租 | MIG/VF/vGPU/独立设备 | 避免仅依赖 Host process sharing |
| 图形/VDI | virtio-gpu、vGPU | 设备/显示生命周期 |
| 研究和验证 | virtio-gpu API remoting | 可观察协议和 Host renderer |
| 跨厂商开发 | virtio-gpu / Vulkan compute | 不依赖单一厂商 runtime |

表中的候选不是保证，仍需 GPU/profile/driver/version 支持矩阵。

## 13. 架构决策树

### 13.1 首先问是否需要独立 Guest OS

- 需要：VM + VFIO/VF/vGPU/virtio-gpu。
- 不需要：container/vendor process sharing。

### 13.2 是否需要完整 CUDA/ROCm runtime

- 需要：整 GPU、VF、vGPU/MIG 的受支持组合。
- 部分需要：native context 或明确支持的 API remoting。
- 只使用 Vulkan：评估 Venus/GFXStream。

### 13.3 是否需要硬件资源隔离

- 强：MIG/VF/vGPU/整卡。
- 中：vendor mediated partition。
- 弱：MPS/time slicing/API shared renderer。

### 13.4 是否需要跨 GPU collective

必须检查：

- P2P topology；
- NVLink/XGMI/PCIe；
- MIG/VF profile 支持；
- NIC affinity；
- collective transport。

### 13.5 是否需要迁移

- 无：多数 GPU 方案可用。
- 进程级：KFD CRIU/应用 checkpoint。
- VM 级：只选择明确支持 migration 的设备/产品组合。

## 14. 评审清单

1. Kubernetes 资源名对应的真实 device view 是什么？
2. MIG、MPS、time slice、VF、vGPU 哪一层负责隔离？
3. Host kernel driver 是否被多个租户共享？
4. device node 通过什么 hook 注入，权限多大？
5. 是否使用 privileged Pod 或 hostPath？
6. GPU memory/HBM、queue、context 和 fault 是否有 quota/隔离？
7. GPU/NIC/NUMA/topology 是否一致？
8. collective 是否验证 P2P 和 transport？
9. Pod 终止后 GPU context/VRAM 是否真正释放？
10. driver/operator 升级如何排空和恢复？
11. telemetry 是否保护 prompt/KV/tensor？
12. 是否把 allocatable 当成物理 capacity？
13. 是否区分硬分区、软配额和 overcommit？
14. 是否有多租户 P99 和故障隔离测试计划？

## 15. 常见误区

### “Pod 拿到 GPU，所以 GPU 已安全隔离”

错。可能只是共享 Host device + process context。

### “DRA/device plugin 实现了 MIG”

错。它们负责分配/描述，MIG/partition 由 GPU vendor stack 创建。

### “MPS 提高利用率，所以适合所有推理多租”

错。小 kernel 并发收益可能伴随 memory、fault、安全与尾延迟风险。

### “time slicing 的 GPU 资源是真分数 GPU”

错。时间共享与空间/硬件分区不同。

### “container 有独立 runtime，就有独立 kernel”

错。容器通常共享 Host kernel 和 Host GPU driver。

### “MIG UUID 存在，就能自动跨节点迁移”

错。assignment、迁移和目的地 profile 是额外能力。

### “Kubernetes 看到 GPU 数等于实际 HBM 容量”

错。分配单位、profile、workspace、fragmentation 和 soft quota 都会改变实际
可服务容量。

## 16. 与其他专题的关系

- VM/设备虚拟化机制：  
  [GPU 虚拟化平台比较](../virtio-gpu/platform-comparison.md)
- 显存和一致性：  
  [GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)
- 执行/调度：  
  [GPU 执行模型、队列、调度与故障隔离](gpu-execution-model.md)
- 安全隔离：  
  [GPU 虚拟化的隔离、安全与故障域](gpu-virtualization-security.md)
- 生命周期和迁移：  
  [GPU VM 生命周期、热插拔、Checkpoint 与迁移](vm-lifecycle-migration.md)
- 性能和可观测性：  
  [GPU 虚拟化性能分析与可观测性](gpu-performance-observability.md)
- NVIDIA MIG/vGPU/MPS：  
  [NVIDIA CUDA、MIG 与 vGPU](nvidia-cuda-vgpu.md)
