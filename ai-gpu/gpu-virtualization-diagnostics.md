# GPU 虚拟化故障诊断 Playbook

## 1. 原则

GPU 故障诊断的核心不是“先看 GPU utilization”，而是找到最早失败或最早停顿的
边界。

```text
symptom
  -> discover expected layer
  -> collect invariant evidence
  -> reject a boundary
  -> proceed downstream
```

本文只提供诊断设计，不执行任何命令、VM、GPU workload 或故障注入。

## 2. 证据纪律

记录：

- workload 和应用期望；
- Guest/Host OS；
- PCI/device topology；
- GPU/driver/firmware；
- QEMU/renderer/runtime；
- profile/MIG/VF；
- IOMMU/ACS；
- exact timestamp；
- 正确性和失败样本；
- 原始日志/trace 的安全保存。

不要：

- 只看 `dmesg` 最后一行；
- 把所有 timeout 归因于 GPU；
- 把 guest `nvidia-smi` 名称当作 passthrough 证据；
- 把一次成功当作资源生命周期正确；
- 上传包含 prompt/tensor/KV 的 trace。

## 3. 分层检查图

```text
L0 business/workload
L1 application/framework
L2 API/runtime/ICD
L3 Guest driver/KMD
L4 Guest transport
L5 VMM/QEMU/device model
L6 Host adapter/renderer/FFI/PF manager
L7 Host driver/IOMMU
L8 GPU firmware/scheduler/engine
L9 memory/cache
L10 display/network/storage
```

早期层失败常伪装成下层性能或 hang。

## 4. 症状 A：Guest 看不到 GPU

### 4.1 先确认设备路径

期望可能是：

- virtio-gpu PCI device；
- VFIO physical function/VF；
- vendor virtual GPU；
- MIG device；
- DRM render node；
- `/dev/kfd`；
- container-injected device。

### 4.2 Virtio-GPU

检查：

1. PCI/MMIO device 是否被 Guest 枚举；
2. virtio feature negotiation；
3. virtio-gpu driver probe；
4. control/cursor/event virtqueue；
5. config `num_capsets`；
6. `GET_CAPSET_INFO/GET_CAPSET`；
7. supported capset bitmask；
8. UMD ICD/loader 是否选择该设备。

常见根因：

- QEMU device variant 错；
- capset 没启用；
- Host renderer init 失败；
- Guest Mesa/ICD 不加载；
- 设备是 2D-only 但应用请求 3D。

### 4.3 VFIO/VF

检查：

- PCI VF exists；
- PF readiness/provisioning；
- VF bound to vfio-pci/vendor；
- IOMMU group viable；
- QEMU config/BAR/IRQ；
- Guest vendor driver probe。

### 4.4 Container

检查：

- K8s resource allocation；
- device plugin/DRA claim；
- CDI/device nodes；
- Host driver mount；
- cgroup/device permission；
- MIG UUID/VF identity。

### 4.5 停止条件

Guest 无设备/无 render node 时，不应继续分析 GPU kernel 性能。

## 5. 症状 B：Device 创建成功，context 失败

### 5.1 Virtio-GPU

检查：

- capset version；
- context init parameters；
- `num_rings`/ring mask；
- QEMU feature；
- renderer create context；
- Host driver/device compatibility。

### 5.2 Venus/GFXStream

- ring blob allocation/map；
- context create command；
- backend object table；
- Host API instance/device creation；
- protocol version。

### 5.3 VFIO/vGPU

- Guest driver firmware/ucode；
- GPU BAR mapping；
- reset state；
- VirtIO vs physical device confusion；
- vGPU profile/guest driver compatibility。

## 6. 症状 C：Submit 卡住

### 6.1 Control packet 到达了吗

采集：

- Guest ioctl/execbuffer；
- virtqueue enqueue/kick；
- QEMU command submit；
- QEMU suspended/choked command；
- renderer entry；
- Host API submit。

如果 QEMU 收到但 renderer 未调用：

- renderer blocked；
- command suspended；
- invalid resource；
- context 不存在。

### 6.2 Ring 通路

Venus/GFXStream:

- Guest tail 是否发布；
- Host head 是否推进；
- ring free space；
- notify 是否发送；
- proxy/render server 是否活跃；
- large payload shmem 是否保活。

### 6.3 KFD

- ring/rptr/wptr；
- doorbell write；
- queue 是否 mapped 到 HQD；
- firmware scheduler/MES；
- event page/IRQ；
- queue priority/eviction。

### 6.4 Host GPU

- Host queue submit；
- GPU scheduler；
- context 是否 hang；
- page fault；
- reset/thermal。

## 7. 症状 D：Fence 不返回

### 7.1 找对应 timeline

`ctx_id + ring_idx + fence_id` 必须映射到：

- Guest virtio dma_fence；
- Host native fence fd；
- Vulkan timeline；
- KFD event；
- vGPU/vendor completion。

### 7.2 区分丢失与慢

- Host work 是否完成；
- native fence 是否 signal；
- renderer callback 是否执行；
- QEMU AIO 是否调度；
- Guest response 是否写入；
- dma_fence 是否 signal；
- waitqueue 是否唤醒。

### 7.3 常见根因

- ring/fence context 不匹配；
- Host callback 线程错误；
- command error 未写 response；
- reset 清掉 pending fence；
- event ID/PASID 错；
- migration 后 stale timeline；
- output fence fd 泄漏/未安装。

## 8. 症状 E：数据错误/旧数据

### 8.1 正确性前三问

1. producer 是否完成？
2. consumer 是否等待了正确同步？
3. producer/consumer 访问的是否同一 backing？

### 8.2 地址

- Guest VA -> GPA；
- Host VA；
- IOVA/IOMMU；
- GPU VA/PTE；
- VRAM/GTT；
- resource ID/offset。

### 8.3 Cache

- DMA direction；
- CPU cache maintenance；
- GPU PTE MTYPE/snoop；
- API barrier；
- explicit fence；
- bounce/SWIOTLB。

### 8.4 Resource

- resource 是否 attach；
- Host backing 是否 import；
- transfer 范围；
- stride/offset；
- dma-buf format/modifier；
- tiling/compression。

### 8.5 典型分类

| 模式 | 可能原因 |
| --- | --- |
| 全部旧数据 | fence/producer 未完成 |
| 局部旧数据 | damage/transfer/cache 范围 |
| stride 错位 | pitch/offset/format |
| 跨进程错 | resource/VA ownership |
| 迁移后错 | PTE/TLB/backing 更新 |
| GPU write 后 CPU 读旧 | coherence/fence |

## 9. 症状 F：离屏正确但窗口黑屏

转到 [GPU 显示、Compositor、WSI 与 VDI](display-wsi-virtualization.md)。

顺序：

1. render target checksum；
2. swapchain acquire；
3. compositor image import；
4. modifier/format；
5. commit/present；
6. scanout/flip；
7. Host backend/remote encode。

离屏成功不能证明 WSI。

## 10. 症状 G：GPU page fault

确认 fault 层：

```text
GPU VA miss
  vs
device DMA IOVA miss / IOMMU PRI
  vs
CPU page fault
```

### 10.1 GPU fault

- VMA/range 是否创建；
- GPU PTE 是否 valid；
- page migration/residency；
- XNACK/retry；
- notifier seqno；
- fault address 权限。

### 10.2 IOMMU fault

- IOAS attachment；
- GPU PTE 产生的 IOVA；
- IOMMU map；
- PASID/Requester ID；
- ATS/ATC/TLB；
- replay/response。

### 10.3 Mixed stack

I/O remoting renderer 可能引入第二层 GPU VM；Guest native VM 正确不等于 Host
renderer 的 VM 映射正确。

## 11. 症状 H：设备 lost/reset

ECC/poison、RAS 和 reset recovery 的完整分类见
[GPU RAS、ECC、Reset 与 AI Job 恢复](gpu-ras-ecc-reset-recovery.md)。

### 11.1 先定范围

- queue；
- engine；
- process/context；
- VF；
- MIG instance；
- whole GPU；
- PCIe hierarchy。

### 11.2 检查

- pre-reset event；
- ECC/RAS/AER；
- queue hang；
- preemption timeout；
- IOMMU/ATS fault；
- VRAM lost；
- source/destination reset；
- stale IRQ/event/fence；
- reinit firmware/context/queue。

### 11.3 API remoting

Host GPU reset 可能 manifest 为 generic proxy error。需要 renderer/Host driver
证据，而不是只看 Guest `device lost`。

## 12. 症状 I：AI framework 无法加载/运行

### 12.1 ABI

- CUDA/HIP library；
- driver/runtime version；
- custom extension compile target；
- KFD/HSA version；
- ICD/loader；
- protocol/capset。

### 12.2 内存

- HBM capacity；
- pinned/Unified memory；
- allocator；
- graph capture；
- IPC/VMM；
- KV cache/workspace。

### 12.3 Collective

- P2P topology；
- NCCL/RCCL provider；
- NIC；
- communicator；
- rank timeout。

详见 [AI Runtime/API 兼容](ai-runtime-api-compatibility.md)。

## 13. 症状 J：性能差/尾延迟高

使用 [GPU 性能与可观测性](gpu-performance-observability.md) 的时间模型。

问：

1. GPU 是否有 queued work？
2. CPU submit 是否慢？
3. ring/virtqueue 是否满？
4. Host renderer 是否 blocked？
5. fence 是否频繁同步？
6. HBM/copy 是否瓶颈？
7. scheduler 是否 oversubscribed？
8. tenant 是否互相干扰？
9. collective slow rank？

### 13.1 单卡快、多卡慢

- P2P 被 ACS/IOMMU 阻止；
- Guest topology 错误；
- VF/MIG peer 限制；
- NCCL transport 选择 Host memory；
- network virtualization。

### 13.2 大 GEMM 快、decode 慢

- launch/fence round-trip；
- graph 未复用；
- small kernel overhead；
- KV cache/moded；
- Host sync。

## 14. 症状 K：Migration/snapshot 失败

按 [生命周期与迁移](vm-lifecycle-migration.md)：

- hard blocker；
- feature/capset/ABI；
- pending GPU work；
- resource/backing；
- GPU VM/PTE；
- context/VRAM；
- fence/event；
- external fd/display；
- destination compatibility；
- rollback/double owner。

不要先调大 destination timeout 掩盖 unsaved state。

## 15. 症状 L：多 VM P2P/collective hang

检查：

- Guest 是否看到 peer device；
- PCIe/ACS/IOMMU；
- VF/MIG profile；
- P2P provider；
- IOMMU domain 交叉；
- network route；
- rank/world；
- topology XML；
- bootstrap；
- NCCL/RCCL debug log。

单卡 ping 或 GPU copy 成功不能替代 collective。

## 16. 各路径最窄的失败边界

| 路径 | 最窄证据 |
| --- | --- |
| VirGL | submit/response/fence + vrend decode |
| Venus | ring head + proxy + Host VkFence |
| GFXStream | ASG seqno + backend fence + QEMU BH |
| Rutabaga | C FFI/component/context table |
| Native DRM | CCMD validation + native BO/fence |
| VFIO | IOMMU group + vendor KMD/firmware |
| SR-IOV | PF provisioning + VF state + IOMMU |
| MIG | profile + assignment + topology |
| vGPU | manager/guest driver/support matrix |
| Container | allocation + device node + Host driver |

## 17. 证据采集清单

### 17.1 Device

```text
lspci topology
PCI IDs/capabilities
IOMMU group
driver/firmware
profile/MIG/VF
```

### 17.2 Guest

```text
dmesg
DRM render nodes
dev/kfd if ROCm
capset/param
runtime error
strace/ioctl category if approved
```

### 17.3 QEMU

```text
command line
device properties
virtio-gpu trace
cmdq/fenceq/renderer state
```

### 17.4 Host GPU

```text
driver/firmware
kernel log/AER/RAS
GPU fault/reset
HBM/utilization
topology/P2P
```

### 17.5 Workload

```text
API/binary version
correctness checksum
latency/throughput
queue depth/fence
queue/fence round trips
```

采集命令必须在获得授权的环境执行，并防止敏感数据泄漏。

## 18. 诊断记录模板

```text
Symptom:
First observed:
Expected behavior:
Actual behavior:
Reproducible inputs:

Guest:
Host:
GPU/firmware:
Virtualization path:
Profile/topology:

Earliest failing layer:
Evidence:
Rejected hypotheses:
Remaining uncertainty:
Next safe check:

Correctness impact:
Security impact:
Tenant scope:
Recovery action:
```

## 19. 常见误诊

### “QEMU 没日志，所以 Guest 没提交”

Guest trace 可先于 QEMU trace。

### “GPU utilization 低，所以 GPU 没问题”

可能 launch/stall/fault/scheduler。

### “fence timeout，所以 GPU hang”

可能 callback、QEMU response 或 Guest wait queue 丢失。

### “dma-buf export 成功，所以 compositor 可显示”

format/modifier/usage/fence 仍需验证。

### “VF reset，其他 VF 正常，所以完全隔离”

可能是偶然时序，不是 reset-domain 证明。

### “Host OOM，肯定 Guest 用太多”

可能是 userptr pin、migration staging、vGPU profile 或 allocator 统计错误。

## 20. 与其他专题

- 源码路径：  
  [四路径横向对照](../virtio-gpu/source-trace-cross-path.md)
- 内存：  
  [GPU 显存与一致性](gpu-memory-semantics.md)
- 执行：  
  [GPU 执行模型](gpu-execution-model.md)
- 安全：  
  [GPU 安全与故障域](gpu-virtualization-security.md)
- 显示：  
  [GPU 显示/WSI/VDI](display-wsi-virtualization.md)
- 性能：  
  [GPU 性能与可观测性](gpu-performance-observability.md)
- 迁移：  
  [GPU VM 生命周期与迁移](vm-lifecycle-migration.md)
