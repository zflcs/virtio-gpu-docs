# NCCL/RCCL Collective 源码跟踪

## 1. 目的与版本

对应总览：[AI GPU 计算栈与虚拟化边界](ai-compute-virtualization.md)。

本文跟踪 NVIDIA NCCL 和 AMD RCCL 如何发现 GPU/NIC topology、建立
communicator、选择 P2P/SHM/NET transport，并构建 ring/tree 等 collective
图。重点说明虚拟化何时改变拓扑判断和完成路径。

固定公开快照：

| 项目 | Commit | 日期 |
| --- | --- | --- |
| NCCL | `3e46c089f4ae4732722a419eb6096aead7e47a30` | 2026-10-08 |
| RCCL | `57e58688f44c77076ad536ef1f6b68741fc6e694` | 2026-01-22 |

## 2. 总体路径

```text
framework / distributed launcher
  -> ncclCommInitRank
  -> commAlloc
  -> peer information all-gather
  -> ncclTopoGetSystem
  -> ncclTopoComputePaths
  -> graph search: ring/tree/NVLS/CollNet/...
  -> transport connect: P2P / SHM / NET / NVLS
  -> per-channel device work graph
  -> collective kernel
  -> completion / error / teardown
```

RCCL 保留大量 NCCL 结构，但增加 AMD XGMI、ASMI/RSMI、ROCm/HIP 和
ROCm-specific topology/tuning。

## 3. Communicator 初始化

固定 NCCL 快照中：

- `ncclCommInitRank()` 在 `src/init.cc:2946`。
- `ncclCommInitRankDev()` 在 `src/init.cc:2851`。
- 异步主体 `ncclCommInitRankFunc()` 在 `src/init.cc:2105`。
- topology/transport 初始化 `initTransportsRank()` 在 `src/init.cc:1163`。

初始化包含：

1. 绑定本地 CUDA/HIP device。
2. 收集 bus ID、host hash、P2P/GDR capability。
3. 用 bootstrap 在不同 rank 间交换 topology XML。
4. 合并各 rank 看到的 GPU/PCI/NIC 图。
5. 构建 path matrix。
6. 搜索 collective graph。
7. 建立 transport connections。
8. 准备 device work、buffer、fence、proxy 和 tuning state。

因此 communicator init 失败不一定是 collective kernel 问题，可能停在
topology discovery、bootstrap 或 transport connect。

## 4. Peer Info 与虚拟化

`fillInfo()` 收集：

- device bus ID；
- host hash；
- P2P/GDR 能力；
- device properties；
- 网络/interconnect 身份。

虚拟化影响：

- VFIO Guest 的 bus ID 是 Guest PCI topology，不是 Host 物理路径；
- 每台 VM 的 `hostHash` 通常只代表 Guest 内核看到的 host；
- VF/SR-IOV、MIG 或 vGPU 的 UUID/bus ID/P2P 能力由 profile/vendor stack
  决定；
- virtio-gpu/API remoting 是否对 NCCL 表现为可用 GPU，取决于 Guest
  CUDA/HIP runtime，而不是 virtio-gpu 自身；
- Guest 内 P2P/NVML/HIP peer 查询必须真的支持目标设备组合。

因此“物理 GPU 在同一 Host”不一定导致 NCCL 选择本地 P2P。

## 5. Topology discovery

固定 NCCL `ncclTopoGetSystem()` 在 `src/graph/topo.cc:1989`：

1. 读取 `NCCL_TOPO_FILE` 或
   `/var/run/nvidia-topologyd/virtualTopology.xml`。
2. 探测 GPU/PCI/CPU/NIC。
3. `ncclTopoFillGpu()` 加入本地 GPU。
4. 导入 Net/CollNet/GIN/RMA 插件设备。
5. bootstrap intra-node all-gather 合并 rank XML。
6. `ncclTopoFuseXml()` 合并节点。
7. `ncclTopoGetSystemFromXml()` 转换为内部 topology。

拓扑节点包括 GPU、PCI bridge/switch、CPU、NIC/NET、NVSwitch/NVS、device；
边包括 NVLink、PCI、C2C、SYS、NET。

RCCL 把 NVLink 映射扩展成 XGMI，并加入 AMD Rome/GIO 拓扑修正。固定
RCCL `src/graph/topo.cc` 中可见：

- `ncclTopoAddXGMI()`；
- `RCCL_TOPO_4P2H_ROME`；
- `parseRome4P2H()`；
- `parse1H16P()`；
- `parse4H4P()`；
- `parseGIOTopos()`。

这说明 collective topology 不只看“有没有网络”，还依赖 CPU/GPU/NIC 的
NUMA、PCIe switch 和 XGMI hive。

## 6. P2P 与 network 选择

PCIe/NVLink/XGMI/UALink/RDMA/CXL 的物理和虚拟化语义见
[AI GPU 互连与虚拟化 Fabric](ai-interconnect-fabric-virtualization.md)。

`ncclTopoComputePaths()` 为节点组合计算路径，并调用：

- `ncclTopoCheckP2p()`；
- P2P transport `canConnect()`；
- SHM transport `canConnect()`；
- network path policy；
- GPU Direct RDMA 检查。

P2P 检查考虑 PCIe/NVLink 距离、`NCCL_P2P_DISABLE/P2P_LEVEL`、NVML peer
状态、MNNVL fabric/clique 和同设备多 rank。

RCCL 的 `RCCL_P2P_NET_DISABLE` 默认值为 1，并可能根据 Rome topology 和
GDR-all 状态选择同节点 network P2P。

虚拟化风险：

- IOMMU/ACS 可能禁用 PCIe P2P；
- VF/MIG/vGPU 是否支持 peer access 依 profile；
- VM 间没有 Guest-visible P2P，collective 必须走网络；
- vGPU/MPT 可能隐藏 Host NVLink/XGMI；
- 网络 plugin 看到的是 vNIC，GDR/rail/PCI path 会改变；
- topology XML 与实际可达路径不一致会造成误选或 hang。

`NCCL_TOPO_FILE` 可用于固定/覆盖 topology，但必须反映真实连通性。

## 7. Graph 和 Channel

NCCL 搜索 Ring、Tree、CollNetDirect、CollNetChain、NVLS、NVLSTree、PAT。
结果不是单一全局 ring，而是 `nChannels` 个 channel。

`ncclTopoPreset()` 填每 channel 的：

- `ring.prev/next`；
- `tree.up/down[]`；
- CollNet 关系；
- NVLS heads。

`ncclBuildRings()` 从 rank 出发沿 `next` 走遍全部 rank，并验证：

- ring 最终回到起始 rank；
- 每个 rank 恰好出现在 ring 中。

`connectTrees()` 设置 parent/child。完成后逻辑 graph 已知，但 transport
connection 还要建立。

## 8. Transport

`initTransportsRank()` 调用：

- `ncclTransportRingConnect()`；
- `ncclTransportTreeConnect()`；
- `ncclTransportPatConnect()`；
- `ncclTransportP2pSetup()`；
- `ncclProxyConnect()`；
- NVLS/CollNet connection。

| Transport | 典型路径 | 虚拟化影响 |
| --- | --- | --- |
| P2P | NVLink/PCIe peer memory | IOMMU/ACS/VF/MIG |
| SHM | 同 Host/VM 内共享内存 | host hash/mmap/HMM |
| NET | IB/RoCE/socket | vNIC、GDR、rail、IP |
| NVLS/multicast | NVSwitch/fabric | 很难跨 VM/partition |
| GIN/RMA | 新网络/offload plugin | capability negotiation |

## 9. RCCL 差异

RCCL 固定提交包含：

- XGMI topology edge；
- AMD SMI/RSMI GPU/link 查询；
- Rome 4P2H、1H16P、4H4P、GIO topology parser；
- optional ROCSHMEM；
- AMD-specific tuning table；
- ROCm/HIP transport 文件。

Guest 看到的是 HIP device 和 Guest topology。Host 物理 XGMI 不会自动透传。
ROCm native context、VF 或 MIG 必须明确支持 peer/topology 查询。

## 10. Collective 执行

`ncclAllReduce` 等调用会：

1. 分类 collective 和消息大小。
2. 按 tuning 选择 algorithm/protocol。
3. 生成 per-channel device/proxy work。
4. GPU kernel 按 ring/tree/NVLS graph 进行 reduce/copy/signal。
5. proxy 处理网络 send/recv/completion。
6. stream/event/fence 报告完成。

虚拟化最敏感的是：

- channel/transport 建立；
- 小消息启动与同步；
- 跨 VM network；
- P2P/GDR；
- proxy/Core 线程唤醒；
- 尾 rank 和尾 link。

## 11. 虚拟化映射

| 机制 | NCCL/RCCL 可见结果 |
| --- | --- |
| VFIO whole GPU | Guest 看见真实 GPU/P2P topology，仍受 IOMMU/ACS/PCI 限制 |
| SR-IOV VF | Guest 看见 VF；peer/XGMI 能力依 profile |
| MIG | Guest 看见 instance/slice；P2P 支持依 profile/platform |
| vGPU | Guest 厂商 device；P2P/topology 由产品决定 |
| virtio-gpu graphics remoting | 通常不提供 NCCL 所需 CUDA/HIP ABI |
| ROCm native context | 目标是 Guest ROCm 使用代理 GPU；P2P 需独立协议 |
| MPS/time slicing | 可能共享物理 GPU，但不是强 VM 隔离 |

## 12. 故障定位

### Communicator init hang

- bootstrap/control network；
- 所有 rank 是否启动；
- host hash/bus ID；
- XML fusion；
- P2P/SHM canConnect；
- network plugin；
- `NCCL_DEBUG=INFO` topology/ring 输出。

### Collective 卡在某个 rank

- PCIe P2P 实际不可达；
- proxy/queue pair 建立失败；
- 错误选择 GDR；
- vNIC MTU/rail 不一致；
- GPU/NIC completion 丢失；
- 一个 VM 的 reset/eviction 影响其他 rank。

### 单卡正常、分布式慢

- P2P 被 IOMMU/ACS 禁用；
- vGPU/VF 隐藏 NVLink/XGMI；
- 跨 VM network 替代本地 fabric；
- topology 变化导致 ring 取代 NVLS/tree；
- 小消息延迟和 tail rank。

## 13. 实验矩阵

当前不执行。未来应交叉测试：

```text
all-reduce/all-gather/all-to-all
  x message sizes
  x ranks per GPU/host
  x P2P on/off
  x bare metal/VFIO/SR-IOV/MIG/vGPU
  x local/cross-VM/RDMA
```

记录 communicator init time、topology XML、transport/channel、bus bandwidth、
P50/P99、租户干扰和 restart/reset 后恢复。

## 14. 源码入口

NCCL：

- `src/init.cc`
- `src/graph/topo.cc`
- `src/graph/paths.cc`
- `src/graph/search.cc`
- `src/graph/connect.cc`
- `src/graph/rings.cc`
- `src/graph/trees.cc`
- `src/graph/xml.cc`
- `src/transport/p2p.cc`
- `src/transport/shm.cc`
- `src/transport/net.cc`
- `src/transport/net_ib/`

RCCL：

- `src/init.cc`
- `src/graph/topo.cc`
- `src/graph/rome_models.cc`
- `src/graph/tuning.cc`
- `src/transport/p2p.cc`
- `src/transport/shm.cc`
- `src/transport/net_ib.cc`
- `src/transport/net_ib_rocm.cc`
- `src/include/amdsmi_wrap.h`

## 15. 检查题

1. NCCL 为什么需要融合 rank topology XML？
2. `hostHash` 在跨 VM collective 中代表什么？
3. 为什么物理 GPU 同 Host 不代表 Guest 使用 P2P？
4. `NCCL_P2P_DISABLE` 与 IOMMU/ACS 限制有何区别？
5. SHM transport 为什么不能替代跨 VM network？
6. NVLS/CollNet 为什么依赖硬件 topology？
7. `ncclBuildRings()` 如何验证 ring 完整？
8. RCCL 为什么需要 Rome/GIO/XGMI parser？
9. 一个 VM 的 GPU reset 如何影响 collective？
10. 如何区分 compute、P2P、network 和 proxy 瓶颈？
