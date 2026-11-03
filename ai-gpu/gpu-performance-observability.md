# GPU 虚拟化性能分析与可观测性

## 1. 目标与纪律

本文解决三个问题：

1. 如何把一个端到端指标拆成 Guest、transport、VMM、host 和 GPU 层；
2. 如何判断瓶颈是 CPU submit、队列等待、GPU compute、HBM、copy、
   synchronization、display 还是 collective；
3. 如何设计不会误导的基准和 trace。

本文不运行 benchmark、VM 或实验。仓库中已有运行结果只在明确注明来源时作为
历史证据，不能当作本次新测量。

性能结论必须同时记录：

- workload 与正确性；
- 软件/硬件版本；
- 配置和拓扑；
- 延迟/吞吐分布；
- 瓶颈证据；
- 可重复性；
- 置信范围和失败样本。

“能运行”“吞吐更高”“GPU utilization 更高”都不是完整结论。

## 2. 先定义测量对象

至少区分：

| 指标 | 问题 |
| --- | --- |
| API latency | 应用一次 API 调用耗时多少？ |
| submit latency | UMD/KMD 到 Host 接受 command 耗时多少？ |
| synchronization latency | submit 到 fence/event 完成耗时多少？ |
| end-to-end latency | 从请求输入到最终输出耗时多少？ |
| throughput | 每秒完成多少 frame/request/step/token？ |
| tail latency | P95/P99/max 是否退化？ |
| GPU busy | engine 是否有可执行工作？ |
| SM/CU busy | compute pipeline 是否真正繁忙？ |
| HBM bandwidth | 数据搬运是否受限？ |
| queue depth | 是否有足够 work 隐藏 CPU/Host 延迟？ |
| migration rate | fault/migration 是否主导？ |
| collective bandwidth/latency | 多卡链路是否成为瓶颈？ |

平均吞吐不能回答交互式推理的尾延迟；GPU busy 不能证明 GPU 在执行有用工作。

## 3. Workload 分类

### 3.1 GPU-bound

特征：

- kernel/draw 执行时间长；
- GPU engine 持续有工作；
- CPU submit 和虚拟化 round-trip 占比较小。

大型矩阵乘、长 compute、部分离线渲染可能属于此类。虚拟化开销可能接近噪声，
但这仍需通过 CPU/GPU 时间线确认。

### 3.2 Launch/submit-bound

特征：

- 单个 kernel/draw 很短；
- 提交频率高；
- GPU 在 work 之间出现空洞；
- CPU、QEMU、renderer 或 fence 线程成为瓶颈。

小 elementwise kernel、推理 decode 和频繁 drawcall 容易暴露虚拟化提交成本。

### 3.3 HBM-bound

特征：

- compute utilization 不高但 HBM bandwidth 高；
- cache miss/occupancy/request rate 与性能相关；
- 增大 batch 或提高 locality 可改善。

### 3.4 Copy-bound

特征：

- H2D/D2H、peer copy 或 format conversion 占主导；
- compute engine 空闲但 copy engine/HBM 繁忙；
- staging、pinned memory 或 WSI 拷贝频繁。

### 3.5 Synchronization-bound

特征：

- 每次 submit 后立即等 fence；
- GPU queue depth 很低；
- CPU/GPU 无法形成流水；
- wait、event、interrupt、callback 延迟显著。

### 3.6 Scheduler-bound

特征：

- work 已提交但长时间未进入 engine；
- queue 多于 HQD slot；
- priority/preemption 频繁；
- aggregate doorbell 或 scheduler firmware 延迟。

### 3.7 Display-bound

特征：

- offscreen benchmark 很好，但对窗口/scanout 帧率差；
- compositor、acquire/release fence、modifier 或 flip 路径受限；
- CPU↔GPU copy 或 Host compositor 占主导。

### 3.8 Collective-bound

特征：

- 单 rank compute 正常但 step time 长；
- collective 等待、链路带宽或 topology 错误；
- rank 间 workload skew 或 P99 straggler。

## 4. 指标分布

### 4.1 不要只报平均值

对每个阶段记录：

- count；
- mean；
- median；
- P90/P95/P99；
- max；
- 标准差/分位数区间；
- timeout/error 数量。

长尾可能来自：

- fence timeout；
- page fault/migration；
- queue 空窗；
- 多租户抢占；
- clock/thermal throttling；
- host scheduler jitter；
- IRQ/thread wakeup；
- collective 慢节点。

### 4.2 Rate 与 latency 分开

```text
throughput = completed_work / wall_time
latency = completion_time - request_time
```

批处理可以在吞吐上升时恶化单请求延迟。推理 serving 要把 batching 条件写清楚。

### 4.3 Utilization 不是效率

```text
busy_fraction = engine_busy_time / observation_window
```

engine busy 可能包含：

- 等待 memory；
- 执行无效/重复 work；
- 因 page fault 阻塞；
- 被低 occupancy 拉长；
- 多租户抢占后的恢复。

必须与有效 ops、HBM bytes、queue wait 和 output rate 联合解释。

## 5. Pipeline 与 Little's Law

稳定异步流水可近似：

```text
concurrency = throughput × latency
```

若每秒提交 2,000 个小 kernel，端到端延迟 2 ms，则系统中平均应有约 4 个
in-flight work。若 queue depth 只有 1，系统无法达到该吞吐。

但 Little's Law 不是性能结论，只用于检查计数一致性：

- 计数窗口是否稳定；
- in-flight 是否包含不同 queue/engine；
- work 是否真正允许并行；
- completion 是否被错误重复计数。

## 6. 端到端时间模型

### 6.1 单次请求

```text
T_request
  = T_frontend
  + T_runtime_encode
  + T_submit_syscall
  + T_guest_kernel
  + T_virtio_or_device_transport
  + T_vmm
  + T_host_proxy_or_parser
  + T_host_runtime_submit
  + T_queue_wait
  + T_gpu_execution
  + T_memory_stall
  + T_completion_transport
  + T_cpu_wakeup
  + T_response
```

### 6.2 稳态 throughput

```text
throughput
  = min(
      application_generation_rate,
      CPU_submit_capacity,
      transport_capacity,
      host_proxy_capacity,
      scheduler_dispatch_capacity,
      GPU_engine_capacity,
      memory_bandwidth_capacity,
      completion_capacity
    )
```

真实系统会有交互项，例如内存压力导致 page fault，fault 又触发 queue
preemption，最终 CPU 和 GPU 同时空闲等待。

### 6.3 Batching

```text
batch_throughput ≈ batch_size / (fixed_submit_overhead + variable_work)
```

增大 batch 通常摊薄固定开销，但增加：

- first-token latency；
- activation/KV memory；
- queue wait；
- cancellation cost。

## 7. 时间来源和时钟域

### 7.1 时钟域

可能同时存在：

- Guest `CLOCK_MONOTONIC`；
- Host OS monotonic；
- QEMU virtual clock；
- GPU timestamp；
- PCIe/network timestamp；
- NIC clock；
- remote rank trace clock。

跨域比较前必须：

- 记录 clock domain；
- 校准 offset/drift；
- 避免用墙钟直接做微秒 delta；
- 保留原始 timestamp；
- 为分布式 trace 建同步点。

### 7.2 Correlation ID

至少关联：

- PID/TID；
- Guest process/context ID；
- command type/opcode；
- fence ID/timeline；
- ring index；
- resource ID；
- queue/event ID；
- Host context ID；
- GPU submission/job ID；
- rank/communicator ID。

同一 ID 在不同层可能改变含义，不能直接相等。

## 8. Virtio-GPU 观测点

### 8.1 Guest 侧

Linux
[`virtgpu_trace.h`](../../linux/drivers/gpu/drm/virtio/virtgpu_trace.h:1) 提供
command queue/response trace，字段包括：

- virtqueue name/index；
- command type/flags；
- fence ID；
- context ID；
- free descriptor 数；
- seqno。

[`virtio-gpu-irq-fence` debugfs](../../linux/drivers/gpu/drm/virtio/virtgpu_debugfs.c:71)
可观察 last/current fence ID。

关键问题：

- queue event 到 response event 的延迟；
- `num_free` 是否接近耗尽；
- fence 是否连续；
- response 是否缺失；
- context 是否长期积压。

### 8.2 QEMU 侧

QEMU
[`trace-events`](../../qemu/hw/display/trace-events:30) 包含 features、resource、
context、submit、map/unmap、flush、suspended command 和 fence 事件。

优先关联：

```text
Guest cmd queue
  -> QEMU ctx_submit
  -> renderer submit
  -> fence ctrl
  -> fence resp
  -> Guest cmd response
```

缺少哪一段，就先定位该边界，而不是直接测 GPU。

### 8.3 Renderer/Host runtime

需要观测：

- parser/dispatch duration；
- Host context/object lookup；
- Host API submit；
- shader/pipeline cache miss；
- resource bind/map；
- fence callback；
- host thread scheduling。

若 Host API submit 到 GPU start 已有明显延迟，问题通常在 Host runtime/queue
而不是 Guest 编码。

## 9. 四条 API Remoting 路径

`vhost-user-gpu` 是 virtio-gpu 的外置后端部署模式，不是新的 API 路径。
其 virtqueue/display socket/fd 传递额外时间边界见
[Vhost-user-gpu 源码跟踪](../virtio-gpu/vhost-user-gpu-source-trace.md)。

### 9.1 VirGL

关键指标：

- Mesa command buffer flush 次数；
- 每次 submit dword 数；
- QEMU payload copy bytes；
- vrend decode time；
- GL state/pipeline cache；
- Host GL completion/fence；
- drawcall 数和 draw 大小。

小 draw 放大 Host GL state 和 QEMU/renderer overhead；大 draw 可能转为
GPU-bound。

### 9.2 Venus

关键指标：

- ring tail/head progression；
- ring free space/wait time；
- notify 次数；
- proxy socket latency；
- `vkr` dispatch time；
- Host VkQueue submission；
- timeline/fence completion。

head 长期落后说明 Host 消费不足；head 推进但 fence 慢说明 Host GPU/runtime。

### 9.3 GFXStream/Rutabaga

关键指标：

- ASG ring utilization；
- encoder/ResourceTracker CPU time；
- QEMU iov copy；
- FFI call；
- GFXStream backend decode；
- Host Vulkan submit；
- QEMU AIO bottom-half latency。

还要区分 command stream 编码失败、Host submit 慢和 WSI copy 慢。

### 9.4 Native DRM context

关键指标：

- Guest UAPI ioctl；
- native protocol command bytes；
- Host native driver submit；
- Guest/Host GPU VM mapping；
- native fence；
- buffer residency/migration。

Native context 可能减少 API 翻译，但 Host queue/driver/memory 成本仍然存在。

## 10. ROCm/KFD 观测点

### 10.1 Queue 状态

采集：

- queue type/AQL/SDMA；
- ring size；
- wptr/rptr；
- doorbell；
- priority/percentage；
- MQD/HQD mapping；
- preemption/reset 状态。

### 10.2 SMI event

KFD
[`kfd_ioctl.h`](../../linux/include/uapi/linux/kfd_ioctl.h:531) 定义：

- VM fault；
- thermal throttle；
- GPU pre/post reset；
- migration start/end；
- page fault start/end；
- queue eviction/restore；
- process start/end。

这些事件能把“GPU 突然慢”与 fault、migration、reset 或 thermal 关联。

### 10.3 Workload 指标

对 ROCm/HIP：

- kernel launch rate；
- queue wait；
- kernel duration；
- occupancy/register/LDS；
- HBM/L2 counters；
- copy engine bytes；
- SVM fault/migration rate；
- stream synchronization。

## 11. VFIO/SR-IOV/MIG/vGPU 观测

### 11.1 VFIO

至少记录：

- IOMMU group/device；
- DMA map/unmap rate；
- interrupt injection；
- VFIO device reset/error；
- DMA dirty rate；
- 是否有 no-IOMMU 或 bounce。

### 11.2 SR-IOV/PF

记录：

- VF identity/profile；
- PF scheduling quantum/priority；
- VF start/stop/reset；
- VRAM quota；
- VF engine/HBM utilization；
- fault 对 sibling VF 的影响。

### 11.3 MIG/vGPU

记录：

- instance/profile；
- partition 内 SM/HBM bandwidth；
- shared engine/PCIe/fabric；
- host manager/scheduler 状态；
- 多租户 interference；
- reset/error scope。

产品遥测的字段名称和粒度可能不同，不能把两个厂商的 “utilization” 当作同一
定义。

## 12. Collective 观测

需要分别记录：

- communicator init；
- topology discovery；
- transport/channel 选择；
- collective size；
- rank arrival skew；
- bus bandwidth；
- latency；
- first/last rank completion；
- retry/hang/timeout。

单卡 GPU 利用率正常，不代表 collective 正常。慢 rank 会让其他 rank 在
collective 中等待。

NCCL/RCCL 的 topology/transport 源码和日志点见
[NCCL/RCCL Collective 源码跟踪](nccl-rccl-source-trace.md)。

## 13. 基准设计

### 13.1 三层 workload

#### Microbenchmark

隔离单个组件：

- command submit；
- fence round-trip；
- H2D/D2H；
- map/unmap；
- 小 kernel launch；
- 一个 collective。

优点是指标清晰；缺点是无法代表真实流水线。

#### Mesobenchmark

代表性内核或整帧：

- GEMM；
- attention；
- video frame；
- 一段 API replay；
- 多 stream 流水。

用于判断持续执行时 CPU/GPU 是否能重叠。

#### Macobenchmark

完整模型、训练 step 或 serving workload：

- prefill/decode；
- 训练 step；
- checkpoint；
- 多卡 collective；
- 多租户并发。

用于最终产品结论。

三层缺一不可：

- micro 能定位；
- meso 能组合；
- macro 能证明用户价值。

### 13.2 Baseline

至少包含：

- 裸机/Host native；
- 当前虚拟化方案；
- 若可能，VFIO/SR-IOV/MIG/vGPU 对照；
- 软件纹理/软件 compute 基线（适用时）。

对照必须保持：

- GPU 型号/profile；
- clocks/power；
- driver/firmware；
- workload 和 batch；
- HBM 容量/频率；
- 输入数据。

### 13.3 Warmup

区分：

- process cold start；
- shader/pipeline compile；
- model load；
- allocator warmup；
- page fault/migration；
- GPU clock ramp；
- cache warmup。

稳态和冷启动分别报告。

### 13.4 运行窗口

若要报告稳态吞吐：

- 明确 warmup 排除规则；
- 固定采样窗口；
- 记录窗口中 throttle/reset/fault；
- 保留失败/超时样本；
- 不用手工删除“看起来异常”的 tail。

### 13.5 重复与随机化

- 多次重复，不只跑一次；
- 交错 A/B，降低热漂移偏差；
- 随机化运行顺序；
- 记录 GPU 温度和功耗；
- 固定或报告 clock policy；
- 对多租户实验使用相同干扰模型。

### 13.6 正确性闸门

性能测试前必须验证：

- 输出数值误差；
- image/frame checksum；
- collective 结果；
- queue/fence 无错误；
- 没有 fallback 到错误 backend；
- 没有意外软件渲染/WSI copy。

更快但走错路径的结果无效。

## 14. 控制变量与噪声

### 14.1 Host 侧变量

- CPU frequency and affinity；
- NUMA placement；
- PCIe topology/AER；
- IOMMU state；
- host load；
- QEMU vCPU pinning；
- renderer thread scheduling；
- huge page/THP；
- host power profile。

### 14.2 Guest 侧变量

- vCPU count/topology；
- Guest kernel/driver；
- Mesa/runtime/ICD；
- allocator；
- queue/ring size；
- fence policy；
- display/compositor；
- VFIO interrupt mode。

### 14.3 GPU 侧变量

- clock/power/thermal；
- ECC state；
- MIG/vGPU profile；
- HBM residency；
- engine contention；
- background GPU daemons；
- peer/fabric traffic。

任何未记录变量都可能解释不同时间的结果差异。

## 15. Bottleneck 决策树

### 15.1 GPU 是否持续有可执行 work？

若否：

1. 应用是否在等前一个 fence？
2. submit syscall/ioctl 是否慢？
3. virtqueue/ring 是否满？
4. QEMU/renderer 是否卡在 parser？
5. Host runtime 是否拒绝/延迟 submit？
6. scheduler 是否有空 HQD slot？
7. 是否发生 fault/migration/reset？

### 15.2 GPU 是否有 work 但 busy 低？

检查：

- HBM/L2 stall；
- low occupancy；
- dependency wait；
- copy engine wait；
- page fault retry；
- preemption/context switch；
- clock throttling。

### 15.3 GPU busy 高但吞吐低？

检查：

- SIMD/SM utilization；
- instruction mix；
- cache hit rate；
- HBM bytes per useful work；
- kernel launch/teardown；
- redundant work；
- collective stragglers。

### 15.4 平均吞吐正常但 P99 差？

优先检查：

- queue waiting；
- fault/migration；
- multi-tenant interference；
- host scheduling jitter；
- interrupt/event wakeup；
- thermal/power；
- rank skew。

### 15.5 offscreen 正常但 present 差？

检查：

- compositor；
- WSI copy；
- acquire/release fence；
- format/modifier；
- scanout/page flip；
- remote display encoding。

## 16. Profiler overhead 与 perturbation

### 16.1 Trace 成本

每事件 trace 可能影响：

- syscall/ioctl 延迟；
- virtqueue 临界区；
- renderer throughput；
- CPU cache；
- log I/O；
- queue backpressure。

尤其要避免在每个小 kernel/draw 上无条件打印大 payload。

### 16.2 Sampling 与计数

GPU counters 需要额外寄存器读写或 profiling 权限，可能：

- serializes work；
- 改变 occupancy；
- 与生产 context 冲突；
- 被 profiler 状态影响。

报告应说明 instrumentation 开关和 overhead。

### 16.3 低扰动顺序

一种合理顺序：

1. 应用/框架遥测；
2. 聚合 counters；
3. 低频 trace；
4. 单次 focused trace；
5. 更细粒度 profiling。

这样可避免第一次就引入过多观测影响。

## 17. 测量安全与数据保护

遥测本身可能泄露：

- 模型权重；
- prompt/KV cache；
- tensor value；
- kernel/shader names；
- memory address；
- peer topology；
- customer workload pattern。

要求：

- 默认不采集大块 payload；
- 对 kernel/path/name 做脱敏；
- trace 文件加密和访问控制；
- 租户隔离；
- 限制 profiler/debug 权限；
- 定义 retention；
- 迁移/崩溃日志同样保护。

性能可观测性必须服从
[GPU 虚拟化的隔离、安全与故障域](gpu-virtualization-security.md) 的威胁模型。

## 18. 实验结果报告模板

### 18.1 Configuration

```text
Workload:
  model/frame/API/graph/batch/sequence

Guest:
  kernel, driver, runtime, Mesa/ROCm/CUDA, queue/fence policy

Host:
  QEMU, renderer, Linux, GPU driver/firmware, vGPU/MIG/VF profile

Hardware:
  CPU, GPU, VRAM/HBM, PCIe/NVLink, NIC, storage

Virtualization:
  virtio/VFIO/SR-IOV/MIG/vGPU, IOMMU/ACS, display/compositor
```

### 18.2 Results

```text
Correctness:
Warmup:
Duration:
Runs:
Latency P50/P95/P99:
Throughput:
GPU/engine utilization:
HBM bandwidth:
Queue depth/wait:
Submit/fence latency:
Fault/migration/reset count:
Tenant interference:
Power/clock/thermal:
```

### 18.3 Attribution

```text
Dominant bottleneck:
Evidence:
Rejected hypotheses:
Remaining uncertainty:
Optimization candidate:
Expected trade-off:
```

## 19. 常见误区

### “GPU utilization 高，所以性能已经最优”

错。可能是低 occupancy、重复 work、memory stall、错误 kernel 或 tenant
争抢把 GPU 拖忙。

### “API 调用很快，所以虚拟化开销小”

错。异步 API 只测了编码/提交入口，没有覆盖 fence/event 和 GPU completion。

### “单个大 GEMM 接近原生，所以 AI 推理接近原生”

错。decode 的小 kernel、KV cache 和 synchronization 可能主导尾延迟。

### “平均 tokens/s 和 frame rate 正常，所以无干扰”

错。要看 P99、抖动、慢 rank 和租户并发。

### “使用 trace 不影响性能”

错。trace/profiler 可能改变 cache、锁、调度和 I/O 行为。

### “CUDA graph/MPS/MIG 一定提升吞吐”

错。收益取决于 launch 比例、workload shape、profile 和共享方式。

### “一次 benchmark 足以得出跨设备结论”

错。需要矩阵化版本、profile、拓扑和多租户条件。

## 20. 性能评审检查表

1. 是否声明了冷启动/稳态和正确性闸门？
2. 是否记录 Guest、Host、GPU、driver、firmware 和 profile？
3. 是否存在裸机/原生或合理软件基线？
4. 是否分别测 API、submit、fence 和 end-to-end？
5. 是否记录 P50/P95/P99，而不只平均？
6. 是否知道 GPU queue 是否有空洞？
7. 是否区分 compute、HBM、copy、sync、scheduler 和 display？
8. 是否检查 ring/virtqueue 满和 QEMU/renderer backpressure？
9. 是否检查 fault/migration/reset/thermal？
10. 是否记录线程 affinity、NUMA、PCIe 和 IOMMU？
11. 是否记录 GPU clock/power/thermal？
12. 是否验证没有 fallback 到不同 backend/copy path？
13. 是否保留 timeout、error 和 tail 样本？
14. 是否评估 trace/profiler overhead？
15. 是否测试多租户 P99 干扰？
16. 是否把 collective 与单卡性能分开？
17. 是否说明结论适用 hardware/profile/version 范围？
18. 是否保护 trace、dump 和 profiler 中的 AI 数据？

## 21. 检查题

1. API latency、submit latency、synchronization latency 和 end-to-end latency
   分别覆盖什么？
2. GPU utilization 高为什么仍可能是 memory-bound 或 scheduler-bound？
3. Little's Law 如何用于检查 in-flight 计数，而不能直接证明瓶颈？
4. 如何用 fence ID、ring index 和 context ID 关联 Guest/Host trace？
5. 为什么 Guest monotonic clock 与 GPU timestamp 不能直接相减？
6. Venus ring head 推进但 fence 慢，应优先查哪一层？
7. QEMU `virtio_gpu_cmd_suspended` 增多意味着什么？
8. KFD SMI 的 fault/migration/event 如何解释异常 tail？
9. 为什么 offscreen 正常但窗口帧率差不能归因于 compute？
10. 如何区分 launch-bound 与 GPU-bound 推理阶段？
11. 什么条件下增大 batch 会提高吞吐但恶化服务延迟？
12. 多租户实验为什么必须同时记录 P50/P99 和隔离 profile？
13. Profiler 如何改变 cache、scheduling 和 synchronization？
14. 为什么 trace/log 也需要按 AI 数据资产保护？

## 22. 与其他专题的关系

- 执行、queue、fence 和 engine：  
  [GPU 执行模型、队列、调度与故障隔离](gpu-execution-model.md)
- 显存、coherence 与 migration：  
  [GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)
- 故障域和 telemetry 安全：  
  [GPU 虚拟化的隔离、安全与故障域](gpu-virtualization-security.md)
- 平台性能模型和方案选择：  
  [GPU 虚拟化平台比较](../virtio-gpu/platform-comparison.md)
- AI workload 与 multi-GPU：  
  [AI GPU 计算栈与虚拟化边界](ai-compute-virtualization.md)
- NCCL/RCCL topology 和 transport：  
  [NCCL/RCCL Collective 源码跟踪](nccl-rccl-source-trace.md)
