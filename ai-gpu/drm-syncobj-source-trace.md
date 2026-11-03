# DRM Syncobj 与 Timeline 同步源码跟踪

## 1. 目标

对应基础文档：

- [DMA-BUF、DMA-FENCE、Reservation 与 Sync File](dma-buf-fence-source-trace.md)
- [GPU 执行模型、队列、调度与故障隔离](gpu-execution-model.md)
- [Virtio-GPU 共用 Linux KMD 源码跟踪](../virtio-gpu/shared-kmd-source-trace.md)

`drm_syncobj` 是 userspace 显式同步对象，主要服务 Vulkan fence/semaphore、
DRM native context、compute queue 和 execbuffer in/out dependency。

它把下面三者连接起来：

```text
userspace handle/timeline point
  <-> drm_syncobj
  <-> dma_fence / dma_fence_chain
  <-> driver queue completion
```

固定源码：

```text
Linux 6b4b543fa18757297ac756a6fe6e64f506b7844a
drivers/gpu/drm/drm_syncobj.c
include/uapi/drm/drm.h
drivers/gpu/drm/virtio/virtgpu_submit.c
drivers/gpu/drm/xe/xe_sync.c
```

本文不运行 syncobj、GPU queue 或 VM 实验。

## 2. Binary 与 Timeline 不是同一种同步

### 2.1 Binary syncobj

核心是一个当前 `dma_fence *`：

```text
NULL
  or
one dma_fence
```

语义：

- reset -> NULL；
- signal -> already-signaled stub fence；
- GPU work completion -> replace current fence；
- wait -> 等待当前 fence；
- 后续 replacement 不影响已经拿到的旧 fence。

### 2.2 Timeline syncobj

timeline 用 u64 point 表示顺序：

```text
point 1 -> point 2 -> point 3 ...
```

每个 point 由 `dma_fence_chain` 组合：

- 当前 driver fence；
- 前面 timeline point 的 fence dependency。

wait point N 表示：

- N 对应的 fence 已出现；
- 且 N 及之前的依赖已完成；
- 不能把不同 syncobj timeline 的 point 数字直接比较。

源码注释明确警告：timeline syncobj 应始终使用 timeline ioctl；混用 binary
ioctl 可能产生错误同步。point 0 可复现 binary 行为。

## 3. Handle、FD 与 Sync File 是三种对象

| 对象 | 作用 |
| --- | --- |
| syncobj handle | 当前 DRM file 内的 u32 对象 ID |
| syncobj fd | 引用完整 persistent syncobj 的匿名 file |
| sync_file fd | 引用“导出时刻的 fence” |

### 3.1 Handle

[`drm_syncobj_get_handle()`](../../linux/drivers/gpu/drm/drm_syncobj.c:598)
把 syncobj 放入当前 DRM file 的 xarray。

handle 只在对应 DRM file namespace 内有意义，不能直接跨进程。

### 3.2 Syncobj FD

[`drm_syncobj_get_fd()`](../../linux/drivers/gpu/drm/drm_syncobj.c:664)创建
anonymous inode fd。

[`drm_syncobj_fd_to_handle()`](../../linux/drivers/gpu/drm/drm_syncobj.c:703)
在另一进程创建新 handle，引用同一 persistent syncobj。

FD import 不会去重 handle：每次导入都创建新 handle/reference。

### 3.3 Sync File FD

[`drm_syncobj_import_sync_file_fence()`](../../linux/drivers/gpu/drm/drm_syncobj.c:728)
把 sync_file 当前 fence 装入 syncobj。

[`drm_syncobj_export_sync_file()`](../../linux/drivers/gpu/drm/drm_syncobj.c:759)
把指定 point 的 fence 导出为 immutable sync_file。

导出之后：

- 后续 reset/signal/replace syncobj；
- 不会改变已导出的 sync_file；
- sync_file 与 syncobj 生命周期各自引用底层 fence。

## 4. Syncobj 创建和 VMA

[`drm_syncobj_create()`](../../linux/drivers/gpu/drm/drm_syncobj.c:556)：

- 分配 `drm_syncobj`；
- kref；
- callback list；
- eventfd list；
- spinlock；
- 可选安装 already-signaled stub fence。

### 4.1 UAPI 创建

`DRM_IOCTL_SYNCOBJ_CREATE` 返回 handle。

`DRM_SYNCOBJ_CREATE_SIGNALED` 不是“绑定某个 GPU fence”，而只是让初始对象
以 signaled stub fence 开始。

### 4.2 Destroy 与 release

handle destroy 从当前 file 的 xarray 移除，eventfd callback 和旧 fence
reference 随之释放。

跨进程 syncobj fd 或另一个 handle 仍持有引用时，底层 object 不会立即销毁。

## 5. Fence replacement 与 timeline point

### 5.1 Binary replace

[`drm_syncobj_replace_fence()`](../../linux/drivers/gpu/drm/drm_syncobj.c:372)
在锁内替换 RCU pointer，然后：

- 唤醒等待旧状态变化的 waiter；
- 更新 eventfd entries；
- 释放旧 fence reference。

### 5.2 Timeline add point

[`drm_syncobj_add_point()`](../../linux/drivers/gpu/drm/drm_syncobj.c:333)：

1. 取得新旧 fence；
2. 如果新 point 不大于已有 point，记录 unordered warning；
3. `dma_fence_chain_init(chain, prev, fence, point)`；
4. 发布新 chain head；
5. 唤醒匹配 waiter/eventfd；
6. GC 旧 chain。

无序 timeline point 不会自动变成正确的顺序语义。应用必须维护 monotonic
submission points。

## 6. 查找 input fence

[`drm_syncobj_find_fence()`](../../linux/drivers/gpu/drm/drm_syncobj.c:436)：

1. 从 handle 找 syncobj；
2. 校验 flags；
3. 取当前 fence；
4. timeline 用 `dma_fence_chain_find_seqno()` 查找 point；
5. point 已 signaled 时返回 signaled stub；
6. 可选 `WAIT_FOR_SUBMIT` 等待 point 真正 materialize；
7. 返回带 reference 的 fence。

`WAIT_FOR_SUBMIT` 可以跨线程：

```text
waiter thread asks point N
submit thread later installs point N
```

但 `WAIT_FOR_SUBMIT` 不能持有 arbitrary kernel locks 睡眠，否则可能与 page fault
等路径死锁。

## 7. Host-side wait

### 7.1 Binary wait

`DRM_IOCTL_SYNCOBJ_WAIT` 接收 handle array。

flags：

- `WAIT_ALL`：全部 signal；
- `WAIT_FOR_SUBMIT`：允许先等待 fence 出现；
- `WAIT_DEADLINE`：设置 deadline hint。

没有 `WAIT_FOR_SUBMIT` 且存在 NULL fence 时返回错误。

### 7.2 Timeline wait

`DRM_IOCTL_SYNCOBJ_TIMELINE_WAIT` 接收 handle array + point array。

增加：

- `WAIT_AVAILABLE`：只等 point 出现，不等 execution completion。

这对 Vulkan timeline semaphore 的 wait-before-signal 语义重要。

### 7.3 多对象等待

[`drm_syncobj_array_wait_timeout()`](../../linux/drivers/gpu/drm/drm_syncobj.c:1196)
为每个 handle/point 建 wait entry：

- all -> 等全部；
- any -> 首个 signaled 返回 index；
- 设置 fence callback；
- 设置 deadline；
- timeout/interrupt 后清理 callback 和 references。

## 8. Eventfd 异步等待

[`drm_syncobj_eventfd_ioctl()`](../../linux/drivers/gpu/drm/drm_syncobj.c:1429)
允许不阻塞等待：

```text
syncobj/timeline point completion
  -> eventfd_signal()
  -> application poll/epoll loop
```

`WAIT_AVAILABLE` 可以在 point 出现时 signal，而不等 GPU 完成。

eventfd entry 持有 syncobj/fence/eventfd references，并注册 dma_fence callback。

## 9. Reset/Signal

### 9.1 Binary reset

`DRM_IOCTL_SYNCOBJ_RESET` 把 current fence 替换为 NULL。

### 9.2 Binary signal

`DRM_IOCTL_SYNCOBJ_SIGNAL` 安装 already-signaled stub。

### 9.3 Timeline signal

`DRM_IOCTL_SYNCOBJ_TIMELINE_SIGNAL` 为每个 handle/point 安装 signaled stub，
保持 point/chained dependency 语义。

这不是 driver 可以无条件伪造 GPU completion 的接口；它表达 CPU 主动 signal。

## 10. Transfer 与 Query

### 10.1 Transfer

`DRM_IOCTL_SYNCOBJ_TRANSFER` 把源 point 的 fence 转移到目标 point。

源码分叉：

- destination point 非 0 -> timeline `add_point()`；
- destination point 0 -> binary `replace_fence()`。

### 10.2 Query

`DRM_IOCTL_SYNCOBJ_QUERY` 返回：

- 最近完成 point；
- 最后 submitted point（可选 flag）。

`LAST_SUBMITTED` 与“已完成”的语义不同。一个 point 可以已经 submit 但尚未
signal。

## 11. Virtio-GPU execbuffer input syncobj

[`virtio_gpu_parse_deps()`](../../linux/drivers/gpu/drm/virtio/virtgpu_submit.c:91)：

1. `copy_from_user` 读取每条 `drm_virtgpu_execbuffer_syncobj`；
2. 校验 flags；
3. `drm_syncobj_find_fence(handle, point, flags=0)`；
4. `virtio_gpu_dma_fence_wait()` 同步等待 input；
5. 若 `RESET` flag，再找 syncobj 保存 reference；
6. submit 完成路径再 reset 这些 input syncobj。

### 11.1 为什么是 submit 前同步 wait

virtio command 不会把 input syncobj 交给 Host renderer 排队；KMD 在将 work
放入 virtqueue 前等待 foreign dependency。

特征：

- input point 尚未出现时通常不会 `WAIT_FOR_SUBMIT`，因为 flags 固定传 0；
- input dependency 的 Host polling/blocking 发生在 Guest KMD；
- BO lock 在 foreign fence wait 之后获取，减少锁等待。

### 11.2 RESET 语义

input syncobj 的 reset 是 userspace 请求的 post-submit 状态归零；需要保持
reference，不能在 parse 到 reset 时提前清除。

## 12. Virtio-GPU execbuffer output syncobj

[`virtio_gpu_parse_post_deps()`](../../linux/drivers/gpu/drm/virtio/virtgpu_submit.c:188)：

- point 0 -> binary output；
- point 非 0 -> 预先分配 `dma_fence_chain`；
- 不允许 output syncobj flags；
- 保存 handle/reference。

[`virtio_gpu_process_post_deps()`](../../linux/drivers/gpu/drm/virtio/virtgpu_submit.c:252)：

```text
submit->out_fence
  -> timeline: drm_syncobj_add_point(point)
  -> binary:   drm_syncobj_replace_fence()
```

关键区别：

- finish submit 时先把 Host completion fence 关联到 output syncobj；
- GPU/host command 实际完成仍由 virtio fence response signal；
- QEMU/Host fence 和 Guest dma_fence context 不可混为一个 timeline。

## 13. Intel Xe sync 对照

[`xe_sync_entry_parse()`](../../linux/drivers/gpu/drm/xe/xe_sync.c:114)
解析 input syncobj：

- binary/timeline point；
- user fence；
- imported syncobj。

[`xe_sync_entry_signal()`](../../linux/drivers/gpu/drm/xe/xe_sync.c:270)：

- timeline output -> `drm_syncobj_add_point()`；
- binary output -> `drm_syncobj_replace_fence()`；
- optional user fence -> add point。

这验证了 syncobj 是跨 GPU vendor driver 的统一 explicit fence ABI，而不是
virtio-gpu 私有对象。

## 14. AMD KFD/DRM 对照

amdgpu compute 和 KFD user queue 也把：

- input syncobj -> dma_fence wait；
- output syncobj -> add/replace；
- KFD event/fence -> dma-fence/sync_file。

ROCm proxy 需要保留这套对象语义，不能只代理 HSA queue command 而丢失
fence timeline。

## 15. Vulkan 映射

| Vulkan 概念 | DRM syncobj 常见表现 |
| --- | --- |
| Binary fence | binary syncobj |
| Timeline semaphore | timeline syncobj + u64 point |
| Wait-before-signal | timeline `WAIT_FOR_SUBMIT/AVAILABLE` |
| Wait in event loop | syncobj eventfd |
| Permanent semaphore | syncobj fd import/export |
| Ephemeral fence | sync_file fd |

Vulkan import/export 还有 payload、temporary/permanent semantics；DRM syncobj
只是 backend 实现的一部分。

不能假设所有 Vulkan semaphore 都能无损转成单个 binary dma_fence。

## 16. 虚拟化边界

### 16.1 Guest binary

Guest UMD 使用的 syncobj handle 只属于 Guest DRM file namespace。QEMU/Host
不可能直接持有同一个 handle。

### 16.2 Host command completion

映射链通常为：

```text
Host renderer/native fence
  -> Host callback
  -> QEMU/virtio fence response
  -> Guest virtio_gpu_fence
  -> Guest syncobj
  -> Guest UMD fence/semaphore
```

Host fence FD 或 pointer 不能直接塞入 Guest handle table。

### 16.3 Timeline

Host renderer 必须保留 Guest ring/point/sequence 语义，或者由 Guest KMD 用
virtio fence 构造自己的 timeline。直接将 Host seqno 暴露给 Guest 会破坏
namespace 和 reset/version 兼容。

## 17. 失败矩阵

| 症状 | 优先检查 |
| --- | --- |
| create unsupported | DRIVER_SYNCOBJ feature |
| binary wait 永远不返回 | fence 未 signal、handle 错、callback 丢失 |
| timeline point 找不到 | point 未 submit / WAIT_FOR_SUBMIT 未设 |
| wait 在错误 point 返回 | binary/timeline ioctl 混用 |
| output point 不出现 | post-deps 未处理、out fence 未 signal |
| any-wait 返回错误 index | first_signaled / array order |
| eventfd 不触发 | point availability、callback/ref |
| sync_file export 后 live update 不同步 | sync_file immutable snapshot 语义 |
| execbuffer 卡住 | input foreign fence、dump fence 未 signal |
| Guest fence 完成但 Host GPU 未完成 | response/fence 映射断链 |
| reset 后旧 waiter 未唤醒 | replacement/callback cleanup |
| cross-process 同步错误 | 错误使用 handle 而不是 syncobj fd |

## 18. 审查清单

### 18.1 Identity

- binary 还是 timeline？
- point 是否 monotonic？
- wait 是 any 还是 all？
- WAIT_AVAILABLE 与 execution complete 是否混用？

### 18.2 Transport

- syncobj 是 handle、syncobj fd 还是 sync_file？
- 跨进程传递了吗？
- export 后旧 point 修改是否应影响 consumer？

### 18.3 Driver

- input fence 在 submit 前 wait 还是进入 queue wait？
- output 使用 add_point 还是 replace_fence？
- reset 何时执行？
- fence error 如何传播？

### 18.4 Virtualization

- Guest/Host fence namespace 是否分离？
- Host seqno 是否错误暴露给 Guest？
- VM reset/teardown 是否 signal error?
- eventfd/callback 是否在对象销毁前注销？

## 19. 推荐源码顺序

1. `drm_syncobj.c` 顶部 Overview。
2. create/destroy/handle/fd。
3. replace_fence/add_point/find_fence。
4. wait/eventfd/query/transfer。
5. `drm.h` syncobj UAPI flags。
6. `virtgpu_submit.c` input/post dependencies。
7. `xe_sync.c` 和 amdgpu user queue 对照。
8. 回到 dma-fence/sync_file 文档。

## 20. 检查题

1. Binary syncobj 与 timeline syncobj 的状态模型差在哪？
2. point 0 为什么可以复现 binary 行为？
3. syncobj handle、syncobj fd 和 sync_file fd 各引用什么？
4. 为什么 sync_file export 是 immutable snapshot？
5. WAIT_AVAILABLE 与 WAIT_FOR_SUBMIT 的区别是什么？
6. any-wait 和 all-wait 的 completion 条件是什么？
7. eventfd wait 如何接入 epoll？
8. virtio-gpu input syncobj 为什么在 submit 前同步等待？
9. output syncobj 何时 add point，何时 replace fence？
10. 为什么 input RESET 不能立即清除对象？
11. Guest syncobj 为什么不能直接持有 Host fence 指针？
12. Host GPU 完成、virtio fence response 和 Guest syncobj signal 的顺序？
13. timeline unordered point 会产生什么行为风险？
14. query LAST_SUBMITTED 为什么不等于 execution complete？
15. VM teardown 时哪些 syncobj/waiter/eventfd 必须清理？

## 21. 与其他专题

- [DMA-BUF、DMA-FENCE、Reservation 与 Sync File](dma-buf-fence-source-trace.md)
- [GPU 执行模型、队列、调度与故障隔离](gpu-execution-model.md)
- [Virtio-GPU 共用 Linux KMD 源码跟踪](../virtio-gpu/shared-kmd-source-trace.md)
- [Native DRM context 源码跟踪](../virtio-gpu/native-drm-source-trace.md)
- [GPU VM 生命周期、热插拔、Checkpoint 与迁移](vm-lifecycle-migration.md)
