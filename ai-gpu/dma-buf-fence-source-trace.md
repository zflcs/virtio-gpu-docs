# DMA-BUF、DMA-FENCE、Reservation 与 Sync File 源码跟踪

## 1. 为什么这是 GPU 虚拟化的共同底座

dma-buf 同时出现在：

- virtio-gpu resource/blob/dma-buf；
- Venus/GFXStream external memory 和 WSI；
- vhost-user-gpu 的 SCM_RIGHTS；
- VFIO PCI dma-buf/P2P；
- DRM PRIME、display/KMS；
- ROCm/KFD dma-buf import/export；
- CUDA/HIP IPС、P2P 和 multi-GPU；
- AI 推理 KV cache、tensor buffer 和跨进程共享。

dma-buf 不是一个 GPU API，而是 Linux 内核跨 driver/device/process 共享
memory 与 asynchronous completion 的基础设施。

固定源码：

```text
Linux 6b4b543fa18757297ac756a6fe6e64f506b7844a
drivers/dma-buf/dma-buf.c
drivers/dma-buf/dma-fence.c
drivers/dma-buf/dma-resv.c
drivers/dma-buf/sync_file.c
drivers/dma-buf/udmabuf.c
include/linux/dma-buf.h
include/linux/dma-fence.h
include/linux/dma-resv.h
```

本文不运行 buffer sharing、P2P、VM 或 GPU workload 实验。

## 2. 一张图

```text
Allocator/exporter
  -> native BO/resource
  -> dma_buf_export()
  -> dma_buf_fd()
  -> dma-buf fd

Importer
  -> dma_buf_get(fd)
  -> dma_buf_attach(device)
  -> dma_buf_map_attachment()
  -> sg_table(DMA addresses)
  -> device DMA
  -> dma_buf_unmap_attachment()
  -> dma_buf_detach()
  -> dma_buf_put()

Synchronization
  -> dma_fence(context, seqno)
  -> dma_resv on dma_buf
  -> sync_file fd for explicit in/out fences
```

## 3. fd 到底代表什么

[`dma_buf_export()`](../../linux/drivers/dma-buf/dma-buf.c:708)
把 exporter 私有对象包装成 `struct dma_buf`，并建立匿名 file。
[`dma_buf_fd()`](../../linux/drivers/dma-buf/dma-buf.c:788)把它安装成 fd。

fd 的跨进程语义：

- `fork`/`SCM_RIGHTS` 复制 fd reference；
- 关闭一个 fd 只减少一个 file reference；
- 只有所有 importer、attachment、mapping 和 fd reference 清理后才 release
  exporter backing；
- `FD_CLOEXEC` 防止 exec 后泄漏；
- fd 是 opaque handle，不描述 pixel format、modifier、stride 或 plane layout。

格式和 layout 必须通过 DRM/KMS/Vulkan/DirectX/媒体 API 的 metadata 单独传递。

## 4. Exporter 与 importer

### 4.1 Exporter

exporter 实现 [`struct dma_buf_ops`](../../linux/include/linux/dma-buf.h:37)：

- attach/detach；
- map_dma_buf/unmap_dma_buf；
- release；
- mmap；
- vmap/vunmap；
- begin/end_cpu_access；
- 可选 pin/unpin；
- 可选 move/notify。

exporter 决定 backing 实际所在位置，并负责所有 importer 共享时的
 migration 或 reallocation。

### 4.2 Importer

importer 用
[`dma_buf_get()`](../../linux/drivers/dma-buf/dma-buf.c:834)验证 fd 类型并取得
`struct dma_buf` reference。

不同 device 必须分别建立 attachment：

```text
same dma_buf
  -> attachment(dev A)
  -> attachment(dev B)
  -> attachment(dev C)
```

attachment 不是 memory 副本。它表示“这个 device 可能访问该 shared object”。

## 5. Attachment 生命周期

[`dma_buf_dynamic_attach()`](../../linux/drivers/dma-buf/dma-buf.c:1032)
创建 attachment，并可选调用 exporter `attach` callback。

[`dma_buf_attach()`](../../linux/drivers/dma-buf/dma-buf.c:1081)是 static
attachment 兼容 wrapper。

动态 importer 可以提供：

- `allow_peer2peer`：是否允许 P2P；
- `invalidate_mappings`：backing 移动时撤销旧 mapping。

### 5.1 Static 与 dynamic attachment

| 类型 | map 行为 | backing move |
| --- | --- | --- |
| static | `dma_buf_map_attachment()`等待 exporter kernel-visible fences | 依赖 pin/lifetime |
| dynamic | importer 自己等待 exclusive fence | exporter 可通过 invalidate 请求重建 mapping |

固定源码把 dynamic attachment 用于支持可撤销、可迁移 mapping 的新模型。

### 5.2 Detach

[`dma_buf_detach()`](../../linux/drivers/dma-buf/dma-buf.c:1097)先从
attachment list 移除，再调用 exporter detach。

顺序必须满足：

```text
停止新 DMA
  -> unmap attachment
  -> detach
  -> dma_buf_put
```

## 6. DMA mapping

[`dma_buf_map_attachment()`](../../linux/drivers/dma-buf/dma-buf.c:1188)
返回 importer device 可用的 `sg_table`：

- exporter 调 `map_dma_buf()`；
- 返回值中的 DMA address/len 按 page size 对齐；
- mapping 存在期间 backing 通常保持 pin；
- mapping 结束必须调 `dma_buf_unmap_attachment()`。

[`dma_buf_unmap_attachment()`](../../linux/drivers/dma-buf/dma-buf.c:1300)
解除 mapping 并按需要 unpin。

### 6.1 为什么不能长期持有 mapping

长期 mapping 会阻止 exporter 迁移 backing，增加 pin 和 memory pressure。
virtio-gpu、VFIO、ROCm、WSI 都可能在 long-lived AI allocation 上放大这个
问题。

### 6.2 显式 wait 责任

源码注释明确：

- static importer 在 map 时会等待 reservation 的 kernel-visible fences；
- dynamic importer 必须先自行等待 exclusive fence；
- map 成功不代表 explicit sync 已完成。

### 6.3 Revocable mapping

[`dma_buf_invalidate_mappings()`](../../linux/drivers/dma-buf/dma-buf.c:1399)
通知 importer 旧 mapping 必须失效。

调用返回时 importer 仍可能访问 memory，因此若要完全停止访问还需：

1. 等待 reservation fences retire；
2. 等 importer 真正 unmap。

这解释了为什么“发一个 invalidation callback”不等于“GPU 已经停止 DMA”。

## 7. CPU access

### 7.1 Kernel CPU access

[`dma_buf_begin_cpu_access()`](../../linux/drivers/dma-buf/dma-buf.c:1536)
和
[`dma_buf_end_cpu_access()`](../../linux/drivers/dma-buf/dma-buf.c:1572)
bracket CPU access：

- exporter cache maintenance；
- 等待 dma_resv 中符合方向的 implicit fences；
- 让文件系统/媒体/GPU 驱动同步 cache domain。

### 7.2 Userspace mmap

[`dma_buf_mmap()`](../../linux/drivers/dma-buf/dma-buf.c:1603)设置 VMA 并调用
exporter mmap。

userspace CPU read/write 必须使用：

```text
DMA_BUF_IOCTL_SYNC(START | READ/WRITE)
  -> CPU read/write
DMA_BUF_IOCTL_SYNC(END | READ/WRITE)
```

该 ioctl 的公开 UAPI 注释限定它主要保证 cache coherence。它不会自动建立
所有 API 层的 queue ordering；explicit fence 仍需应用或 importer 管理。

## 8. DMA-FENCE

### 8.1 身份

[`struct dma_fence`](../../linux/include/linux/dma-fence.h:70)核心字段：

- context；
- seqno；
- signaled/error/time；
- refcount；
- callbacks。

`context` 表示独立 timeline；`seqno` 只在同一 context 内比较先后。

```text
(context A, seqno 10) and (context B, seqno 20)
  -> 不能仅凭 seqno 判断谁先完成
```

### 8.2 Signal

[`dma_fence_signal()`](../../linux/drivers/dma-buf/dma-fence.c:488)：

- 设置 signaled；
- 记录 timestamp；
- 运行 callback；
- 唤醒 waiter；
- 不可从 signaled 回到 unsignaled。

### 8.3 Wait

[`dma_fence_wait_timeout()`](../../linux/drivers/dma-buf/dma-fence.c:525)
先 enable signaling，然后同步等待或 timeout。

[`dma_fence_add_callback()`](../../linux/drivers/dma-buf/dma-fence.c:698)
提供异步 completion callback。

fence 是 reference-counted object。发起 wait/callback 的一方必须保证自己持有
足够长的 reference。

### 8.4 Fence array/chain

多个 fence 可以通过：

- `dma_fence_array`：全部完成后 aggregate fence 完成；
- `dma_fence_chain`：交织 timeline chain；
- unwrap helper；

组合。sync_file 可能包含 container fence，而不是单个硬件 queue fence。

### 8.5 不可把用户可控未来 fence 随便塞进内核

Linux 文档明确指出：future/proxy/userspace-controlled indefinite fence 与
kernel dma-fence dependency 混用可能形成循环：

```text
kernel memory-management fence -> userspace fence
userspace fence -> kernel fence
```

因此不要为了虚拟化 proxy 随意把 Guest 可无限延迟的 completion 伪装成普通
dma_fence。可恢复 page fault/long-running compute 尤其要重新审查
implicit-fence 假设。

## 9. DMA Reservation

### 9.1 Usage

[`enum dma_resv_usage`](../../linux/include/linux/dma-resv.h:71)：

| Usage | 含义 |
| --- | --- |
| `KERNEL` | kernel memory management，总是必须等待 |
| `WRITE` | implicit write dependency |
| `READ` | implicit read dependency |
| `BOOKKEEP` | 不参与 implicit sync，常用于显式同步提交 |

查询更严格的 usage 会包含更低层级 fences。源码注释中的
`KERNEL < WRITE < READ < BOOKKEEP` 描述包含/过滤顺序，不是时间顺序。

### 9.2 Add fence

[`dma_resv_add_fence()`](../../linux/drivers/dma-buf/dma-resv.c:287)：

- 取得 fence reference；
- 同 context 新 fence 可以替换更早 fence；
- signaled old fence可被清理；
- list 由 RCU 保护。

### 9.3 Wait

[`dma_resv_wait_timeout()`](../../linux/drivers/dma-buf/dma-resv.c:677)
按 usage 等待所有匹配 fence。

`dma_resv_test_signaled()` 只测试状态，不保证之后保持 signaled；对象仍可能被
新提交写入。

## 10. Sync File

[`sync_file_create()`](../../linux/drivers/dma-buf/sync_file.c:65)把一个
dma_fence/aggregate fence 包装成 userspace fd。

[`sync_file_get_fence()`](../../linux/drivers/dma-buf/sync_file.c:103)从 fd
取回 fence reference。

### 10.1 In/out fence

- out-fence：driver -> userspace，表示 buffer 的写/使用完成后 signal；
- in-fence：userspace -> driver，driver 必须等待后才可访问 buffer。

### 10.2 UAPI

[`include/uapi/linux/sync_file.h`](../../linux/include/uapi/linux/sync_file.h)
提供：

- `SYNC_IOC_MERGE`；
- `SYNC_IOC_FILE_INFO`；
- deadline；
- driver/timeline/status/timestamp 信息。

sync_file fd 可以跨进程传递。它表达 explicit synchronization，不自动向
dma-buf reservation 增加 fence。

### 10.3 Implicit vs explicit

| 机制 | 依赖存储 | 谁负责等待 |
| --- | --- | --- |
| implicit | dma_resv | kernel/driver 路径 |
| explicit | sync_file/fd + API command | userspace/API/driver |

dma-buf UAPI 也提供 export/import sync-file ioctl，把两者桥接；使用前必须查
当前内核和 exporter 支持。

## 11. UDMABUF

`udmabuf` 将 memfd/shmem pages 包装为 dma-buf：

[`udmabuf_create()`](../../linux/drivers/dma-buf/udmabuf.c:350)：

1. 校验 memfd 和 seals；
2. 按 offset/length 解析 backing；
3. pin folios；
4. 建立 `udmabuf` object；
5. `dma_buf_export()`；
6. 返回 dma-buf fd。

[`udmabuf_pin_folios()`](../../linux/drivers/dma-buf/udmabuf.c:304)
建立 page/fn backing；
[`get_sg_table()`](../../linux/drivers/dma-buf/udmabuf.c:131)为 importer
生成 DMA sg。

### 11.1 使用边界

- backing 来自 sealed memfd，不是任意 CPU 地址；
- nodefault mappings 可提供多条 memfd ranges，具体约束由 UAPI 定义；
- page pin 与 memfd seal 共同保证 lifetime；
- 仍需要 exporter/importer dma mapping 和 fence。

udmabuf 常用于让 Guest-compatible shmem 变成 Host dma-buf，而不是完整 SVM
实现。

## 12. DMA Heaps、格式和 modifier

`/dev/dma_heap/*` 提供 userspace allocation：

- system heap：linear/cacheable；
- CMA heap：physically contiguous；
- CoCo shared heap：加密 VM 中可被设备访问的 shared/decrypted memory。

dma-buf fd 仍不包含图形 metadata。WSI/导入必须另传：

- pixel format；
- width/height；
- plane offsets；
- strides；
- modifier；
- usage；
- acquire/release fence。

公开 userspace 文档要求所有 allocator/importer 对 format+modifier 做能力交集，
不能只检查 fd type。

## 13. 映射到 virtio-gpu

### 13.1 Blob 与 dma-buf

```text
Guest GEM/BO
  -> dma_buf fd
  -> QEMU/renderer resource import
  -> Host API/Vulkan/DRM BO
```

或反向 Host 分配：

```text
Host renderer BO
  -> dma-buf
  -> Guest resource/mmap
```

fd 只共享 object；`VIRTGPU_MAP`、blob mmap 或 WSI import 才决定 userspace
如何访问。

### 13.2 Fence

virtio-gpu 至少有：

- virtio control response fence；
- Guest dma_fence/syncobj；
- Host API/native fence；
- display acquire/release。

Host renderer callback 完成时应 signal Guest dma_fence；control response 只
证明 command 被处理。

### 13.3 vhost-user

vhost-user-gpu 通过 SCM_RIGHTS 传 dma-buf fd：

- 内核 object 共享；
- userspace pointer 不共享；
- fd close 各自管理引用；
- display update 仍需要 sync。

## 14. 映射到 VFIO/P2P 与 AI

VFIO DMA-BUF/P2PDMA 把 VFIO region 的 physical pages 转成 dma-buf physical
vector，供 peer device import。

AI 多 GPU tensor sharing 需要：

- dma-buf object lifetime；
- exporter/importer attachment；
- GPU peer access/IOMMU/ACS；
- correct fences；
- topology 与 NUMA；
- process/VM ownership。

“dma-buf import 成功”仍不代表：

- GPU P2P 可用；
- 能 zero-copy 访问；
- layout 可被目标 engine 使用；
- collective 运行正确。

## 15. AI workload 的典型生命周期

```text
allocate/import tensor storage
  -> virtual address map
  -> queue read/write with fences
  -> reuse across iterations
  -> checkpoint/export
  -> stop writers
  -> invalidate mappings or wait
  -> unmap/detach/release
```

大模型场景：

- 权重长期占用，mapping 不应无意阻止 eviction；
- KV cache 高频增长/复用；
- activation/workspace 短生命周期，适合 pool；
- collective peer mapping 需要长期但可撤销；
- checkpoint 必须 drain GPU writes；
- reset/hot-unplug 必须处理 importer mapping 和 stale fences。

## 16. 失败矩阵

| 症状 | 优先检查 |
| --- | --- |
| fd 不能 import | fd 类型、O_CLOEXEC、权限、正确 subsystem |
| import 成功但图像错乱 | format/modifier/stride/plane metadata |
| DMA map 失败 | device 可达性、placement、pin、IOMMU |
| CPU 读到旧数据 | begin/end_cpu_access 或 DMA_BUF_IOCTL_SYNC |
| GPU 读到旧数据 | acquire fence/explicit sync/implicit resv |
| fence 永不完成 | signal path、context/seqno、callback、driver ownership |
| 同 context fence 乱序 | seqno/ring/timeline 使用错误 |
| 不同 context fence 比较错误 | 错误跨 context 比较 seqno |
| P2P map 失败 | dynamic attachment、allow_peer2peer、provider/ACS |
| unmap 长时间卡住 | pin、GPU 访问、TLB、exporter move |
| fd 泄漏 | O_CLOEXEC/SCM_RIGHTS/错误路径 |
| VM 迁移后 fd 失效 | 跨 VMM/Host 重建 object 和 fences |

## 17. 审查清单

### 17.1 Object ownership

- exporter 是谁？
- 每个 importer 是否独立 attach？
- 谁持有 dma-buf/fence/attachment reference？
- 谁关闭跨进程 fd？

### 17.2 Address mapping

- DMA map 返回的是 Host DMA address 还是 IOVA？
- 是否长期 pin？
- backing move 是否需要 revoke？
- 每个 device 的 mapping 是否独立？

### 17.3 Synchronization

- implicit 还是 explicit？
- dma_fence context/seqno 是什么？
- sync_file 中是单 fence 还是 array/chain？
- CPU access 和 GPU access 是否 bracket？
- reset/error 是否 signal fence with error？

### 17.4 Virtualization

- Guest fd 如何成为 Host dma-buf？
- 是否经过 QEMU/renderer/vhost-user？
- fence ID 如何映射？
- VM teardown 是否先停止 importer 再 unmap？
- migration 如何重建 dma-buf/dma_resv？

## 18. 源码阅读顺序

1. `Documentation/driver-api/dma-buf.rst`。
2. `dma-buf.c` 的 export/get/attach/map/mmap/CPU access。
3. `dma-fence.c` 的 signal/wait/callback。
4. `dma-resv.c` 的 usage/add/wait。
5. `sync_file.c` 与 UAPI。
6. `udmabuf.c`。
7. dma-heaps 文档。
8. 回到 Venus/GFXStream/WSI、VFIO、ROCm 和 virtio-gpu 的具体 import/export 路径。

## 19. 检查题

1. dma-buf fd 是否携带 format、modifier、stride 和 plane offset？
2. `dma_buf_get()` 与 `dma_buf_attach()` 各取得什么 reference/object？
3. 为什么同一个 dma-buf 需要对每个 device 单独 map？
4. attachment map 成功后 backing 是否一定永久有效？
5. dma_fence context 与 seqno 如何定义 timeline？
6. 为什么不同 context 的 seqno 不能比较大小？
7. dma_resv 的 KERNEL/WRITE/READ/BOOKKEEP 如何过滤 fence？
8. sync_file 的 in/out fence 分别由谁创建、谁等待？
9. implicit 与 explicit sync 如何在 dma-buf UAPI 桥接？
10. `DMA_BUF_IOCTL_SYNC` 能保证什么，不能保证什么？
11. `dma_buf_invalidate_mappings()` 返回后为什么还必须等待 fence/unmap？
12. udmabuf 为什么不等于任意 userptr registration？
13. virtio-gpu 的 control response 为什么不等于 GPU dma-fence signal？
14. VFIO dma-buf import 成功为什么仍不能推出 P2P 可用？
15. AI checkpoint/VM migration 需要保存或重建哪些 dma-buf/fence 状态？

## 20. 与其他专题

- [GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)
- [GPU 执行模型、队列、调度与故障隔离](gpu-execution-model.md)
- [GPU 显示、Compositor、WSI 与 VDI](display-wsi-virtualization.md)
- [PCIe、IOMMU、ATS/PASID 与 GPU P2P](pcie-iommu-ats-p2p.md)
- [Virtio-GPU Userptr 与 ROCm 社区动向](../virtio-gpu/userptr-rocm-compute.md)
- [Vhost-user-gpu 源码跟踪](../virtio-gpu/vhost-user-gpu-source-trace.md)
- [QEMU VFIO 源码跟踪](qemu-vfio-source-trace.md)
