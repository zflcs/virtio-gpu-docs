# Virtio-GPU Userptr 与 ROCm 社区动向

具体 renderer 源码跟踪见
[ROCm compute 源码跟踪](rocm-source-trace.md)。

全库固定的 commit、证据等级和上游复核规则见
[GPU 虚拟化文档证据登记册](../EVIDENCE_REGISTER.md)。

本文以 **2026-10-29** 为快照，分析 Linux virtio-gpu `userptr` 和 AMD
ROCm native context 的公开提案、ABI 与未解决问题，重点解释 virtio-gpu
如何从图形渲染扩展为 Host GPU compute 能力共享，以及为什么 compute 比普通
渲染更需要新的内存与同步语义。

先说结论：

1. 当前 Linux 主线工作树**已经合入** resource blob、context init 和 blob
   alignment，但**没有合入**本文讨论的 userptr UAPI/驱动实现。
2. Linux userptr 系列已迭代到 RFC v9（2026-09-24），仍在评审，不能把
   Patchwork 上存在的补丁视为可用稳定 ABI。
3. QEMU 最新公开 userptr/ROCm 系列仍是 RFC v4（2026-01-07），其 feature
   编号已经落后于之后的 Linux v9，全栈 ABI 尚未对齐。
4. virglrenderer ROCm native context MR
   [`!1568`](https://gitlab.freedesktop.org/virgl/virglrenderer/-/merge_requests/1568)
   到 2026-10-29 仍为 opened、需要 rebase；其中仍有 capset/flag/`map_info`
   等实现问题。
5. 方向很重要：它尝试让 Guest ROCm/HIP 生态通过 virtio-gpu 使用 Host
   AMD GPU，而不是要求 Guest `VFIO` 独占整张 GPU。但它仍是 API/libhsakmt
   proxying，不是 PCI passthrough，也不自动获得 SR-IOV 或 MIG 式隔离。

## 1. 当前可验证基线

### 1.1 当前主线 UAPI

[`virtgpu_drm.h`](../../linux/include/uapi/drm/virtgpu_drm.h:95) 当前公开：

```text
VIRTGPU_PARAM_3D_FEATURES          = 1
VIRTGPU_PARAM_CAPSET_QUERY_FIX     = 2
VIRTGPU_PARAM_RESOURCE_BLOB        = 3
VIRTGPU_PARAM_HOST_VISIBLE         = 4
VIRTGPU_PARAM_CROSS_DEVICE         = 5
VIRTGPU_PARAM_CONTEXT_INIT         = 6
VIRTGPU_PARAM_SUPPORTED_CAPSET_IDs = 7
VIRTGPU_PARAM_EXPLICIT_DEBUG_NAME  = 8
VIRTGPU_PARAM_BLOB_ALIGNMENT       = 9
```

[`virtio_gpu.h`](../../linux/include/uapi/linux/virtio_gpu.h:31) 当前公开 feature：

```text
VIRTIO_GPU_F_VIRGL           = 0
VIRTIO_GPU_F_EDID            = 1
VIRTIO_GPU_F_RESOURCE_UUID   = 2
VIRTIO_GPU_F_RESOURCE_BLOB   = 3
VIRTIO_GPU_F_CONTEXT_INIT    = 4
VIRTIO_GPU_F_BLOB_ALIGNMENT  = 5
```

当前 blob flags：

```text
VIRTGPU_BLOB_FLAG_USE_MAPPABLE      0x0001
VIRTGPU_BLOB_FLAG_USE_SHAREABLE     0x0002
VIRTGPU_BLOB_FLAG_USE_CROSS_DEVICE  0x0004
```

源码搜索 `userptr` 只在 execbuffer 的通用用户地址辅助函数中出现，没有
`virtio_gpu_userptr_create()`、`VIRTGPU_PARAM_USERPTR` 或
`VIRTGPU_BLOB_FLAG_USE_READONLY`。这证明本地 2026-10-25 内核还没有合入提案。

### 1.2 当前 ROCm 路径状态

| 组件 | 最新公开状态 | 判定 |
| --- | --- | --- |
| Linux virtio-gpu userptr | RFC v9，2026-09-24 | 未合入 |
| Linux KFD virtio 需求 | 转向 amdgpu DRM SVM 设计讨论 | 未稳定 |
| QEMU userptr/ROCm | RFC v4，2026-01-07 | 落后于 Linux v9 |
| virglrenderer ROCm MR | `!1568` opened，2026-10-29 更新 | 未合入 |
| ROCm runtime | 前端 virtio 支持部分进入 ROCm 仓库 | 生态集成推进中 |

Patchwork、GitLab MR 和 ROCm 仓库不是同一条发布流水线。一个组件存在代码，
不代表其他组件已经实现兼容 ABI，更不代表发行版内核和 QEMU 可以直接启用。

## 2. 为什么 compute 需要 userptr

图形 API remoting 通常让 Host renderer 分配多数 GPU resource。大量 compute
程序则依赖一个不同假设：应用分配一块普通进程内存，CPU 和 GPU 都直接使用它，
不需要每次操作先显式复制到另一块“GPU 专用 staging buffer”。

ROCm/HIP 中常见的模型包括：

- pageable host memory；
- pinned host memory；
- coarse-grained coherence；
- fine-grained coherence；
- GPU SVM/ATS 或 GPU page-table mapping；
- userptr allocation。

本节需要先纠正名称。不同系统里的 “userptr” 至少有三种含义：

1. **DRM ioctl 的 userptr**：驱动 pin 一个用户 VA 范围，把页给 GPU 使用。
2. **VIRTGPU blob userptr 提案**：Guest KMD pin Guest 用户页，作为
   `CREATE_BLOB` backing 发给 Host。
3. **libhsakmt/ROCm userptr 对象**：ROCm 栈中 CPU/GPU 共享同一 storage 的
   allocation 语义。

三者相关但不相同。本文讨论的是第 2 类，并把它接到第 3 类 compute 语义。

现有 virtio-gpu shmem blob 也会把 Guest 页转成 mem entry，但它通常先在 Guest
创建 shmem backing，再建立 virtio backing。提案的目标是让 compute 应用已有
mapping 直接成为 blob backing，避免额外 Guest shmem allocation 和一次
memcpy。它**不能保证整个系统没有拷贝**：QEMU、DMA API、IOMMU 或 SWIOTLB
仍可能产生 bounce buffer。

### 2.1 为什么 compute 不能只照搬渲染路径

渲染路径主要传递 draw、pipeline、render target 和 WSI；compute 栈还依赖
长时间存活的 CPU/GPU 共享 allocation、队列/事件、SVM 和细粒度同步。

| 问题 | 典型图形 remoting | ROCm compute |
| --- | --- | --- |
| 主要 Guest API | GL/Vulkan 图形命令 | HIP/ROCr/HSAKMT |
| 内存模型 | texture/buffer/WSI image | host memory、device memory、userptr、SVM |
| 内存生命周期 | 以 frame/资源为中心 | 以进程 allocation 和指针为中心 |
| 同步 | render/present fence | queue、event、semaphore、coherence |
| Host 执行 | GL/Vulkan renderer | Host libhsakmt/KFD + amdgpu |
| 典型失败 | 黑屏、绘制错误 | 读到旧数据、页错误、pin 泄漏、进程崩溃 |

把 ROCm 只映射到 Venus/Vulkan compute 可以覆盖部分 kernel，但无法完整替代
ROCm 库、HIP runtime、KFD/HSA 对象模型和大量 AI/HPC workload。v9 因此选择
新增 ROCm capset 和 HSAKMT-oriented proxy，而不是仅复用图形 capset。

### 2.2 virtio-gpu 如何利用 Host GPU 的 compute 能力

virtio-gpu 本身既不执行 GPU kernel，也不实现 HIP。它提供四类基础能力：

1. **设备与能力发现**：ROCm capset 表示 Host renderer 能代理 compute
   协议。
2. **控制传输**：Guest libhsakmt frontend 把对象/队列/事件/HSA 操作编码成
   virtio-gpu control commands。
3. **内存共享**：blob、dma-buf 和拟议 userptr 把 Guest storage 暴露给 Host
   renderer/driver。
4. **完成通知**：virtio fence/timeline 把 Host GPU completion 返回 Guest。

真正的 compute 执行链在 Host renderer 之后：

```text
Guest HIP application
  -> ROCr / libhsakmt virtio frontend
  -> virtio-gpu ROCm commands
  -> QEMU
  -> virglrenderer ROCm context
  -> Host libhsakmt
  -> Host /dev/kfd + amdgpu KFD
  -> GPU compute queue
  -> Host completion -> virtio fence -> Guest
```

Host libhsakmt 使用 Host 的 KFD/amdgpu 驱动管理 GPU queue、event、VM 和
memory。Guest 看到的是 virtio-gpu，不是物理 GPU；Host 物理 GPU 的 compute
engine 被多个 Guest/context 共享或由 Host 调度器调度。

如果平台使用硬件 SR-IOV/MIG/厂商 vGPU，则由对应机制提供资源切分和调度。
ROCm native context 提案本身仍属于软件 mediated path，不能自动获得这些硬件
隔离属性。

## 3. Userptr v9 的完整提案

### 3.1 版本时间线

公开系列从 2024-12 的 RFC 发展到 2026-09 的 v9：

| 时间 | 版本 | 主要变化 |
| --- | --- | --- |
| 2024-12-20 | RFC v1 | blob userptr resource 初版 |
| 2025-02-28 | v1 | 加入 probing、object、interval tree、MMU notifier |
| 2025-03-21 | v2 | 简化 userptr 管理 |
| 2025-11-12 | 新拆分系列 | HSAKMT context + userptr |
| 2026-01-15/16 | v4/v5 | 独立 compute 系列，探测与接线逐步明确 |
| 2026-09-17/18 | v6/v7/v8 | UAPI 大改，去掉独立 userptr feature，加入 readonly |
| 2026-09-24 | RFC v9 | ROCm capset 改到 10，保留 `USE_READONLY`，待 virtio spec 更新 |

早期版本考虑过 MMU notifier，后来去掉了复杂的 movable-userptr 设计。v9
选择对给定 VA 范围做 long-term pin；这意味着应用不能把这批页当成可随时
迁移或 munmap 的普通匿名内存。

### 3.2 v9 UAPI

Linux v9 在 `drm_virtgpu_resource_create_blob` 末尾追加：

```c
__u64 userptr;
```

约定：

- `userptr == 0`：走原有 blob 创建路径；
- `userptr != 0`：创建由该 Guest 用户 VA 起始地址 backing 的 userptr blob；
- 只有 `VIRTGPU_BLOB_MEM_GUEST` 可与非零 `userptr` 组合；
- 地址和长度都按页对齐；
- userspace 必须先 probe `VIRTGPU_PARAM_USERPTR`；
- 不支持这个字段的旧内核不能静默当成 shmem blob。

新增 getparam：

```text
VIRTGPU_PARAM_USERPTR        = 10
VIRTGPU_PARAM_BLOB_READONLY  = 11
```

新增 blob flag：

```text
VIRTGPU_BLOB_FLAG_USE_READONLY = 0x0008
```

新增 virtio feature：

```text
VIRTIO_GPU_F_BLOB_READONLY
```

v9 不再通过独立 `VIRTIO_GPU_F_RESOURCE_USERPTR` wire feature 选择 userptr；
非零 `userptr` 字段选择 Guest KMD 路径。wire 上新增的是 readonly 语义：
Host 必须承诺不会写该 resource。Guest 只有在 Host 宣告
`VIRTIO_GPU_F_BLOB_READONLY` 时，才会去掉 `FOLL_WRITE`。

### 3.3 v9 capset

virtio UAPI 提议：

```text
VIRTIO_GPU_CAPSET_ROCM = 10
VIRTGPU_DRM_CAPSET_ROCM = 10
```

ID 10 是为避开已经出现的 ID 7-9 冲突。v9 cover letter 明确指出：
gfxstream/rutabaga 生态已经占用了部分未统一到该内核头文件中的 ID，因此不能
继续使用 8。

capset 只表示“协议家族”；userptr feature/param 是另一个维度。不能因为
存在 `CAPSET_ROCM` 就推断 userptr 一定可用，也不能因为普通 blob 可用就
推断 ROCm capset 可用。

## 4. Guest KMD 的真实生命周期

v9 新增 `virtio_gpu_object_userptr`。核心流程如下：

```text
userspace 分配普通进程 mapping
  -> DRM_VIRTGPU_RESOURCE_CREATE_BLOB
       blob_mem = GUEST
       flags    = [READONLY]
       userptr  = mapping start
       size     = mapping length
  -> Guest KMD 校验 feature、对齐、长度和权限
  -> pin_user_pages_fast(..., FOLL_LONGTERM)
  -> pages -> sg_table
  -> 必要时 dma_map_sgtable()
  -> mem entry = DMA address (或物理地址)
  -> VIRTIO_GPU_CMD_RESOURCE_CREATE_BLOB
  -> QEMU / renderer
  -> RESOURCE_UNREF 完成后 unpin
```

关键代码职责：

1. **地址校验**：非空、页对齐、加法不溢出、`access_ok()`。
2. **长期 pin**：`FOLL_LONGTERM`；普通 writable blob 加 `FOLL_WRITE`，
   readonly blob 不加。
3. **内存统计**：计入 `mm->pinned_vm`，受 `RLIMIT_MEMLOCK` 和
   `CAP_IPC_LOCK` 约束。
4. **scatterlist**：把不连续的 pinned pages 变成 SG table。
5. **DMA 映射**：只在 `virtio_gpu_use_dma_api()` 的路径执行。writable 使用
   `DMA_BIDIRECTIONAL`，readonly 使用 `DMA_TO_DEVICE`。
6. **Backing 发送**：`virtio_gpu_mem_entry.addr/length` 来自 DMA SG 或
   physical SG。
7. **Hibernation restore**：重新建立映射并发送 `CREATE_BLOB`，不走 shmem
   恢复路径。
8. **Release**：对象仍被 Host 使用时不能提前 unpin；要等
   `RESOURCE_UNREF` response。
9. **Dirty 标记**：writable user pages unpin 时按写回规则标记 dirty。
10. **PRIME 限制**：userptr object 的 dma-buf export 当前显式返回
    `-EINVAL`，因为导出会把用户页生命周期传播到当前驱动控制之外。

这不是简单的“保存一个指针”。其真正对象是：

```text
Guest mm + pinned pages + SG table + optional DMA map
  + Guest resource ID + Host resource lifetime
```

## 5. Host 路径与“零拷贝”争议

### 5.1 提案中的期望路径

概念上：

```text
Guest ROCm application memory
  -> Guest virtio-gpu userptr blob
  -> QEMU iovec / memory mapping
  -> virglrenderer ROCm backend
  -> Host libhsakmt + /dev/kfd
  -> Host GPU page table references the backing
```

如果 Host GPU 真能直接对 Guest 页的 DMA 地址执行访问，并且 Guest/Host
缓存属性与 fence 契约都正确，那么计算数据不需要再复制到另一块 Host VRAM
staging buffer。这正是提案有价值的部分。

### 5.2 不能称为无条件零拷贝

v9 cover letter 已明确删除“零拷贝”的泛化表述，只说避免第二次 Guest shmem
allocation 和 memcpy。原因包括：

- QEMU 仍要构造 Host 映射/iovec；
- DMA API 可能使用 SWIOTLB bounce buffer；
- IOMMU 映射只是地址翻译，不等于 CPU cache 一致性；
- GPU 页表可能把页映射到 GTT/system memory，而不是 VRAM；
- 迁移、pin/unpin、GPU page fault 可能引入隐藏复制或同步；
- libhsakmt/HIP 的 coarse/fine-grained coherence 仍要求正确的 GPU cache
  attributes 和 API-level synchronization。

准确说法应是：**消除了 Guest 侧额外 shmem staging 的一次分配和 memcpy，
但不保证跨 QEMU、DMA 层和 GPU cache 的端到端零拷贝。**

### 5.3 社区提出的 coherence 问题

v9 评审中，Akihiko Odaki 针对 `dma_map_sgtable()` 提出：

- HIP coarse-grained coherence 只在同步点之后保证可见；
- fine-grained coherence 要求执行期间持续一致；
- 如果 DMA map 需要 bounce buffer 或显式 cache maintenance，不能仅靠
  GPU 页表 MTYPE/SNOOPED 补足；
- 建议使用 `DMA_ATTR_REQUIRE_COHERENT`，让无法提供连续 user-memory
  coherence 的平台直接拒绝。

v9 diff 仍使用 `dma_map_sgtable(..., attrs=0)`，说明这项反馈尚未在 v9
代码中落实。Dmitry Osipenko 也指出应继续使用更合适的 DMA API 语义，而不是
把所有平台都当作 coherent。

这类问题对专家很重要：API 的 “zero-copy” 目标不是由一个 flag 证明的，而是
要同时证明地址映射、DMA coherence、GPU cache policy、同步点和 fault
handling。

## 6. v9 仍未关闭的错误路径

v9 已修复一部分早期问题：

- pin 失败时清空 `pages`，避免 double free；
- userptr 只能配 `GUEST` blob；
- readonly 必须有 Host feature；
- 去掉不可达的 transfer DMA-sync 分支；
- restore 路径记录 bounce buffer 可能性。

但 2026-09-27 的公开评审仍指出：

> userptr 创建已经 pin 页、计入 memlock、发送 resource create 后，如果
> 后续 `drm_gem_handle_create()` 或 UUID assignment 失败，旧错误路径直接
> `drm_gem_object_release()`，不会调用 GEM `.free`，导致 pinned pages、DMA
> map、memlock 和 Host resource 泄漏。

作者回复会在下一版修复。v9 本身尚未修复这条路径。`RESOURCE_UNREF`
completion 的设计也因此不仅是性能优化，还是页生命周期正确性的核心。

另一个评审点是 `pinned_vm` 负值检查行为不清晰，作者表示后续移除。主机页大小
与 Guest 页大小可能不同，`blob_alignment` 校验由通用 `verify_blob()` 和
userptr 创建共同保证。

## 7. QEMU 提案与当前 kernel v9 的错位

QEMU 最新公开系列是 RFC v4：

- [`1/3` blob `USE_USERPTR`](https://patchwork.kernel.org/project/qemu-devel/patch/20260107075030.3453924-2-honghuan@amd.com/)
- [`2/3` `VIRTIO_GPU_F_RESOURCE_USERPTR`](https://patchwork.kernel.org/project/qemu-devel/patch/20260107075030.3453924-3-honghuan@amd.com/)
- [`3/3` ROCm capability](https://patchwork.kernel.org/project/qemu-devel/patch/20260107075030.3453924-4-honghuan@amd.com/)

它提出：

```text
-device virtio-gpu-gl,...,userptr=on,hsakmt=on
```

并计划把 Guest CREATE_BLOB mem entries 转成 renderer 的 iovec。

但它是 2026-01 的 ABI：

- 使用 `VIRTIO_GPU_F_RESOURCE_USERPTR = 5`；当前 Linux 已把 feature 5
  定义为 `BLOB_ALIGNMENT`。
- 使用 wire `VIRTIO_GPU_BLOB_FLAG_USE_USERPTR`；v9 主要保留
  `USE_READONLY`，非零 Guest UAPI `userptr` 字段选择路径。
- 使用 HSAKMT/ROCm 命名和 capset 方案早于 v9 的 ID 10 决定。

因此不能把 2026-01 QEMU 补丁直接配 2026-09 kernel v9。需要等待/制作与最终
kernel UAPI 和 virtio spec 对齐的新 QEMU 系列。

QEMU v4 还直接删除了旧的 `nr_entries > 16384` 限制。安全 review 不能只接受
“支持更多分散页”，必须确认总字节数、整数溢出、内存配额和 malicious Guest
资源上限都有替代保护。

## 8. Linux KFD 与 DRM SVM 方向

早期 ROCm 全栈还需要额外 KFD 补丁。2026-10 作者在 userptr review 中说明：

- KFD maintainer 希望把 virtio 所需能力纳入新的 amdgpu DRM SVM 设计，而不是
  继续堆 KFD 专用 patch；
- amdgpu VM maintainer 参与考虑 virtio requirements；
- 过渡方案可在 userspace 使用 `mremap`，但会有性能损失；
- 相关 amd-gfx 讨论从 2026-08 开始推进。

这表示社区方向已从“给 KFD 打一个虚拟化旁路”转向“让 SVM/统一 GPU VM
框架原生表达跨 VM 地址与 userptr 生命周期”。这是比 userptr 单个 ioctl 更
深层的设计变化。

对学习者的含义：ROCm 虚拟化不能只读 virtio-gpu KMD；还必须理解 amdgpu
VM、KFD SVM/HMM、GPU page fault、process isolation 和新 DRM SVM helper。

## 9. virglrenderer ROCm native context MR

### 9.1 架构

MR `!1568` 的实际方向不是把 `/dev/kfd` ioctl 原样穿过 virtio，而是把 HSAKMT
相关对象和操作转成 renderer command：

```text
Guest HIP / ROCr / libhsakmt front end
  -> HSAKMT-oriented virtio control commands
  -> QEMU virtio-gpu
  -> virglrenderer rocm context
  -> Host libhsakmt
  -> Host /dev/kfd + amdgpu
  -> GPU
```

提案新增 `src/rocm/`，覆盖：

- device 初始化、node/GPU 属性；
- HSAKMT context；
- memory object、host memory、userptr、dma-buf、scratch；
- queue create/update/destroy；
- event create/set/reset/query；
- query 接口；
- VM address-space interval manager；
- requests/response shmem 和 protocol definitions；
- virglrenderer resource type `VIRGL_RESOURCE_VA_HANDLE`；
- blob `VIRGL_RENDERER_BLOB_FLAG_USE_USERPTR`。

Host 的 `vhsakmt_device_get_blob()` 为 blob 创建 Host object；VA-based
resource 不再经过二次 map，而是把已协商的 virtual address 直接作为 resource
identity。这里的 VA 映射表不是 GPU MMU 本身，而是 renderer 对 Host/guest
对象和 VA 区间的软件账本。

### 9.2 当前可见缺口

2026-10-29 MR 快照存在以下公开问题：

1. **未 rebase**：MR 仍不能直接合并。
2. **capset 编号不一致**：MR 新增常量仍使用
   `VIRGL_RENDERER_CAPSET_HSAKMT = 8`，Linux v9 已改为
   `CAPSET_ROCM = 10`。
3. **初始化 flag 冲突**：MR 定义 `VIRGL_RENDERER_USE_ROCM (1 << 14)`，
   当前主线同一头文件已将 bit 14 用于 `VIRGL_RENDERER_USE_GUEST_VRAM`。
   如果直接合并，会让两个含义共用 flag，属于必须修正的 ABI/实现缺陷。
4. **`get_blob` map_info 未初始化**：2026-10-29 最新评论指出
   `vhsakmt_device_get_blob()` 不初始化 `blob->map_info`，后续
   `virgl_renderer_resource_get_map_info()` 会失败。
5. **CI 缺位**：reviewer 要求启用 compile CI。
6. **未完成全栈 ABI 对齐**：kernel v9、QEMU v4、renderer MR 对 capset、
   feature、blob flag 的设计不在同一时间基线。

MR 页面中的性能结果属于提案方测量，当前仓库没有对应的独立环境、原始
trace、统计方法和复现记录。本文只把它们当作“候选可行性证据”，不作为
架构优劣结论。

### 9.3 为什么不是 `amdgpu` native context 的简单扩展

AMD 的两条 native path 面向不同 kernel userspace ABI：

- Mesa/RADV 使用 `/dev/dri/renderD*` 和 DRM amdgpu UAPI；
- ROCm/libhsakmt 使用 `/dev/kfd` 和 HSA/KFD ABI。

ROCm 不是 libdrm 的上层。`libhsakmt` 是 ROCm low-level runtime，拥有独立
object、queue、event 和 VM 语义，因此不能简单让现有 amdgpu native context
代理 RADV 的 ioctl 就得到完整 HIP。

## 10. 与图形 native context 的对照

ROCm/HIP/ROCr/libhsakmt/KFD 与 CUDA、Vulkan 和图形 API 的兼容层次见
[AI Runtime、API 与虚拟化兼容性](../ai-gpu/ai-runtime-api-compatibility.md)。

| 维度 | 普通 native DRM / AMDGPU | ROCm compute native context |
| --- | --- | --- |
| Guest 入口 | Mesa RADV/AMD common shim | HIP/ROCr/libhsakmt front end |
| Host API | libdrm_amdgpu / amdgpu DRM UAPI | libhsakmt / KFD + amdgpu |
| Guest-visible device | 同一 virtio-gpu，DRM capset 6 | 同一 virtio-gpu，ROCm capset |
| 命令语义 | GEM、VA、CS submit、fence | HSA memory、queue、event、SVM/userptr |
| 内存挑战 | GEM/dma-buf/VA | userptr coherence、pinning、SVM |
| 当前成熟度 | 多个上游 native backend | AMD 提案，未合入 |
| 安全模型 | Host renderer 代理，不是 VFIO | Host libhsakmt 代理，不是 VFIO |

两者都可能在 GPU-bound workload 中接近原生，但 CPU-bound、memory-lifetime
heavy 或 synchronization-heavy workload 的开销必须单独测量。

## 11. 如何判断社区消息是否代表“已经可用”

采用以下五级证据：

| 等级 | 证据 |
| --- | --- |
| E0 | 博客、路线图、厂商宣传 |
| E1 | 公开 patch series / draft MR |
| E2 | Patchwork/MR 最新版本，有实现和 review 回复 |
| E3 | 所有相关组件 ABI 对齐，维护者接受，CI 通过 |
| E4 | 合入各自主线并出现在发布内核/QEMU/renderer/ROCm |
| E5 | 独立平台测试、故障/安全/迁移与性能数据可复现 |

截至本文快照：

- userptr KMD：E2；
- QEMU userptr/ROCm：E1/E2，且 ABI 落后；
- virglrenderer ROCm：E2，有开放缺陷；
- ROCm runtime：部分组件进入其仓库，但还不足以构成 E3/E4 全栈；
- 生产可用性：未达到 E5。

专家在评审新动态时必须标出“提案”“merge-ready”“merged”“released”和
“production-proven”的区别。

## 12. 下一步应跟踪的具体信号

1. Linux v10 是否解决 GEM handle/UUID failure cleanup 和 DMA coherence。
2. `VIRTIO_GPU_F_BLOB_READONLY` 和 `USERPTR` UAPI 是否进入 virtio spec。
3. v9 的 capset 10 是否成为最终编号。
4. QEMU 是否发布与 kernel v9/v10 和 spec 对齐的新系列。
5. virglrenderer MR 是否 rebase、修复 flag 冲突、`map_info` 和 CI。
6. KFD 需求是否真正落到 amdgpu DRM SVM helper，而不是长期私有旁路。
7. 大页、NUMA、dma-buf export、hot-unplug、hibernation、live migration 和
   error recovery 是否有测试。
8. 是否出现不依赖 bounce buffer 的平台和有公开方法的独立性能对比。

## 13. 检查题

1. 为什么主线的 `RESOURCE_BLOB` 存在，不代表 userptr 已经存在？
2. 为什么非零 `userptr` 必须配 `VIRTGPU_BLOB_MEM_GUEST`？
3. `FOLL_LONGTERM` 对应用 munmap、内存迁移和 `RLIMIT_MEMLOCK` 有什么影响？
4. readonly blob 为什么可以不要求 `FOLL_WRITE`，但它仍可能需要同步？
5. 为什么 v9 删除了“zero-copy”这个绝对表述？
6. `dma_map_sgtable` 成功后，Guest 和 GPU 是否必然 cache coherent？
7. userptr resource 为什么暂不允许 PRIME export？
8. GEM handle 创建失败为什么不能让普通 `drm_gem_object_release()` 直接结束？
9. 为什么 QEMU RFC v4 不能直接搭配 Linux v9？
10. virglrenderer 的 ROCm context 为什么需要独立 capset，而不是复用
    `CAPSET_DRM`？
11. 为什么 ROCm native context 仍不是 VFIO passthrough？
12. 如果 GPU-bound 应用在 proposal benchmark 接近原生，能否据此断言小粒度
    同步 workload 也接近原生？

## 14. 外部原始入口

- Linux v9 cover：
  <https://patchwork.kernel.org/project/dri-devel/cover/20260924095556.1326164-1-honghuan@amd.com/>
- Linux v9 UAPI：
  <https://patchwork.kernel.org/project/dri-devel/patch/20260924095556.1326164-3-honghuan@amd.com/>
- Linux v9 KMD：
  <https://patchwork.kernel.org/project/dri-devel/patch/20260924095556.1326164-4-honghuan@amd.com/>
- QEMU RFC v4：
  <https://patchwork.kernel.org/project/qemu-devel/patch/20260107075030.3453924-3-honghuan@amd.com/>
- virglrenderer MR `!1568`：
  <https://gitlab.freedesktop.org/virgl/virglrenderer/-/merge_requests/1568>

外部状态会变化。引用时应记录访问日期、系列版本、patch ID 和具体 commit，
不要只保留一个可变网页标题。
