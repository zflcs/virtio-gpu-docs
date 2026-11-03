# Virtio-GPU 共用 Linux KMD 源码跟踪

## 1. 范围

VirGL、Venus、GFXStream 和 Native DRM context 共用 Guest Linux
`virtio_gpu` DRM driver。本文跟踪它们共用的：

- probe/feature negotiation；
- control/cursor queues；
- context；
- GEM/resource/blob；
- attach backing/mem entries；
- execbuffer；
- virtio/virtqueue submit；
- completion/fence/syncobj/event；
- object destroy；
- suspend/hibernate/reset。

各路径的 API encoder、Host renderer 和 fence 细节仍见对应专题。

源码基线：Linux `6b4b543fa18757297ac756a6fe6e64f506b7844a`。

本文不运行 VM 或 GPU workload。

## 2. 总分层

```text
Guest UMD
  -> DRM UAPI
  -> virtio_gpu KMD
  -> virtio split virtqueue
  -> QEMU virtio-gpu
  -> renderer/device backend
```

KMD 是 transport/object/fence 层，不是 GL/Vulkan/GFXStream 语义层。

## 3. Probe 与初始化

### 3.1 Probe

[`virtio_gpu_probe()`](../../linux/drivers/gpu/drm/virtio/virtgpu_drv.c:74)：

1. 检查 modeset 参数；
2. `drm_dev_alloc()`，使用 virtio parent device；
3. 处理 PCI quirk；
4. 设置 max segment size；
5. `virtio_gpu_init()`；
6. `drm_dev_register()`；
7. `drm_client_setup()` 创建 framebuffer console。

Virtio device 本身不分配 DMA ops；DRM device 使用实际 parent 的 DMA 属性。

### 3.2 Features

`virtio_gpu_init()` 读取：

- `VIRTIO_F_VERSION_1` mandatory；
- `VIRTIO_GPU_F_VIRGL`；
- `VIRTIO_GPU_F_EDID`；
- `VIRTIO_F_INDIRECT_DESC`；
- `VIRTIO_GPU_F_RESOURCE_UUID`；
- `VIRTIO_GPU_F_RESOURCE_BLOB`；
- host-visible shared memory region；
- `VIRTIO_GPU_F_CONTEXT_INIT`；
- `VIRTIO_GPU_F_BLOB_ALIGNMENT`；
- scanout/capset count。

Feature 缺失会改变 UAPI 可用性和 UMD 行为，但不会改变基础 2D device。

### 3.3 Queues

`virtio_gpu_init()`：

- 初始化 `ctrlq`/`cursorq` lock、work、waitqueue；
- 分配 DRM fence context；
- 初始化 fence list；
- capset cache；
- config changed work；
- object free work/list；
- hibernation restore list；
- host-visible DRM MM allocator；
- `virtio_gpu_find_vqs()`；
- `virtio_gpu_alloc_vbufs()`；
- modeset init；
- `virtio_device_ready()`。

主要 queues：

| queue | 内容 |
| --- | --- |
| control | resource/context/submit/fence/display commands |
| cursor | update/move cursor |

每个 request 用 `virtio_gpu_vbuffer` 持有 command/response payload、object refs、
callback 和 fence。

## 4. Virtqueue 生命周期

### 4.1 vbuf

`virtio_gpu_vbuffer` 至少有：

- command header/payload；
- optional response buffer；
- object array；
- response callback；
- sync-for-cpu marker；
- virtqueue sequence number。

vbuf 在 device 使用期间必须有独立生命周期，不能指向用户栈或已释放 resource。

### 4.2 Enqueue

`virtio_gpu_queue_fenced_ctrl_buffer()` 将：

- command；
- optional output data；
- response；

组成 SG list。无 indirect 时可能占多个 descriptor；有 indirect 时合并为 indirect
element。

如果 queue free descriptor 不足：

- 先 notify device；
- 等 `ack_queue`；
- device unplug 时 `vqs_released` 唤醒等待并返回 `-ENODEV`。

### 4.3 Fence emit

只有 descriptor 空间已确认后，源码才分配 fence seqno 并写 command header。

原因：vbuf 已确定能入队，才能把 fence ID 与真实 submit 绑定。

Object 引用还会加入 fence，保证 BO 至少存活到 completion。

### 4.4 Response reclaim

[`virtio_gpu_dequeue_ctrl_func()`](../../linux/drivers/gpu/drm/virtio/virtgpu_vq.c:238)
在 workqueue：

1. disable callback；
2. drain virtqueue used buffers；
3. re-enable callback 并重查 race；
4. 先对 `DMA_FROM_DEVICE` transfer 做 `dma_sync_sgtable_for_cpu()`；
5. 发布 `from_host_pending=false`；
6. trace response；
7. error 分类；
8. fence signal；
9. per-command callback；
10. 释放 object refs/vbuf。

### 4.5 Sync-before-signal

源码注释明确：必须在 signal 任何 fence 之前完成 CPU-visible DMA sync。

因为 `virtio_gpu_fence_event_process()` 也会 signal 同 context 更早 fences；如果
同步放在逐 command loop 中，另一 waiter 可能先被唤醒并读 bounce buffer。

这是“completion signal 顺序必须覆盖 memory visibility”的具体案例。

## 5. Context

### 5.1 Implicit/explicit context

VirGL 可由 `DRM_VIRTGPU_MAP`/其他 path 隐式创建 context。

Venus/GFXStream/native context 通常使用
`DRM_IOCTL_VIRTGPU_CONTEXT_INIT`：

- capset ID；
- number of rings；
- poll ring mask；
- debug name。

### 5.2 Context init ioctl

`virtio_gpu_context_init_ioctl()`：

- 拒绝 context 已创建后的重复 init；
- 校验 param type/value；
- 对 capset/ring mask 做范围检查；
- 分配 per-ring fence contexts；
- 保存 `num_rings/ring_idx_mask`；
- 发送 context create。

### 5.3 Ring index

`EXECBUFFER` 的 `ring_idx`：

- 必须小于 `num_rings`；
- 需要 `base_fence_ctx`；
- 决定 dma_fence context 和 completion timeline；
- 对 poll mask ring 建立 DRM event。

Guest KMD 不解析每个 ring 对应的 Host GPU engine；它只保持 timeline 独立。

## 6. GEM/Resource

### 6.1 GEM create

`virtio_gpu_object_create()`：

1. size round up page；
2. `drm_gem_shmem_create()`；
3. allocate global resource ID；
4. 初始化 object params；
5. 建立 SG/mem entries；
6. 根据 blob/virgl/普通 resource 选择 create command；
7. attach backing；
8. 保存 non-virgl params 到 restore list。

### 6.2 Resource ID

`hw_res_handle` 是 Guest 与 virtio device 共享的 resource ID。

它不是：

- Host renderer pointer；
- dma-buf fd；
- GPU VA；
- GEM handle。

### 6.3 Blob

`VIRTGPU_BLOB_MEM_GUEST`：

- Guest shmem/GEM backing；
- SG mem entries。

`HOST3D`：

- Host renderer backing；
- Guest 通常通过 hostmem window map 或 resource handle 引用。

`HOST3D_GUEST`：

- 两侧 backing 描述；
- 具体 share/export 由 flags/backend 决定。

### 6.4 Attach backing

`virtio_gpu_object_shmem_init()`：

- 获取 shmem SG table；
- DMA API path 写 `sg_dma_address()`；
- non-DMA path 写 `sg_phys()`；
- 构造 `virtio_gpu_mem_entry[]`。

`virtio_gpu_cmd_resource_attach_backing()` 将 entry array 作为 command payload。

### 6.5 Object free

`virtio_gpu_free_object()`：

- 若 Host resource 已创建，发送 `RESOURCE_UNREF`；
- callback 最后 `virtio_gpu_cleanup_object()`；
- 不能立即释放 GEM backing；
- 要先确保 device/Host 不再引用。

## 7. DRM UAPI

Main ioctls:

- `GETPARAM`；
- `GET_CAPS`；
- `CONTEXT_INIT`；
- `RESOURCE_CREATE/INFO`；
- `RESOURCE_CREATE_BLOB`；
- `MAP/UNMAP`；
- `TRANSFER_TO_HOST/FROM_HOST`；
- `EXECBUFFER`；
- `WAIT`；
- `RESOURCE_UNREF`。

UAPI 是 capset-neutral。不同 UMD 在 command payload/ring 中编码不同协议。

## 8. Execbuffer

### 8.1 Order

[`virtio_gpu_execbuffer_ioctl()`](../../linux/drivers/gpu/drm/virtio/virtgpu_submit.c:478)：

1. 要求 3D feature；
2. 校验 flags/ring_idx；
3. 创建 context（若需要）；
4. 分配 output fence/event；
5. 复制 BO handle list；
6. 复制 command bytes；
7. 创建 output sync_file/fd；
8. 解析 post/input syncobj；
9. 等待 foreign input fence；
10. lock BO reservation；
11. command submit；
12. notify virtqueue；
13. 安装 output fence fd。

### 8.2 Command copy

`vmemdup_user()` 使 KMD 拥有内核 private command bytes。后续 QEMU 还会再复制
guest payload；renderer/UMD protocol 不能依赖用户 buffer 永久有效。

### 8.3 BO lock

BO list lock/reservation 防止：

- command 引用期间 backing 被迁移/销毁；
- object fence 与 free race；
- 不同 context 并发改 resource state。

### 8.4 Output fence conditions

当需要：

- output fence fd；
- output syncobj；
- BO list；
- ring fence event；

时分配 out fence。无同步需求的 submit 可减少 fence 分配。

### 8.5 Submit command

`virtio_gpu_cmd_submit()` 填：

```text
VIRTIO_GPU_CMD_SUBMIT_3D
ctx_id
size
command SG
response SG
```

vbuf object array 保活 command 引用 resource。

### 8.6 Notify

`virtio_gpu_notify()` 只执行必要的 virtqueue kick；具体 vq mapping/notification
由 Virtio core 决定。

“notify 返回”只表示 device 被通知，不表示 command 完成。

## 9. Fence 与 syncobj

### 9.1 Fence allocation

`virtio_gpu_fence_alloc()`：

- fence context = base context + ring_idx；
- seqno 暂为 0；
- 在 vbuf 成功入队时 emit。

### 9.2 Fence completion

`virtio_gpu_fence_event_process()`：

- 按 response fence ID 查找；
- 只处理匹配 dma_fence context；
- signal 该 fence 及同 context 更早 fence；
- 发送 DRM event；
- 释放 fence。

### 9.3 多 ring 并行

不同 ring_idx 有独立 fence context。把不同 physical queue 合并到同一 context
会破坏并行/完成语义。

### 9.4 Syncobj/FD

输入：

- foreign dma_fence wait；
- input syncobj wait flags/points。

输出：

- output syncobj point；
- sync_file fd。

用户必须区分 host syncobj timeline 和 Virtio fence timeline。

syncobj binary/timeline、handle/fd、sync_file、wait/eventfd/query 的公共实现见
[DRM Syncobj 与 Timeline 同步源码跟踪](../ai-gpu/drm-syncobj-source-trace.md)。

GEM object/handle、fake mmap offset 与 PRIME import/export 的公共实现见
[DRM GEM 与 PRIME 源码跟踪](../ai-gpu/drm-gem-prime-source-trace.md)。

## 10. Cursor queue

cursor queue 单独处理：

- cursor image resource；
- hotspot；
- position/move；
- immediate ack。

cursor queue 不依赖 primary render fence；cursor work 的 completion 与 draw
completion 是两个路径。

## 11. Interrupt/config

Device config change 通过 `config_changed_work` 更新：

- display info；
- capset cache；
- events。

`GET_DISPLAY_INFO/GET_EDID` response 让 connector/mode 更新。

Event/poll ring 通过 DRM event 与 fence signal 关联。

## 12. Reset

KMD reset 需要：

- 停止/等待 ctrl/cursor work；
- detach unused vbuf；
- signal/清理 pending fence；
- destroy Host resources；
- reset display/scanout；
- 重新 ready device。

QEMU device reset also clears cmd/fence queues; Guest driver must tolerate device
generation loss.

## 13. Suspend/hibernation

### 13.1 Freeze

`virtgpu_freeze()`：

1. DRM mode config suspend；
2. hibernation 时 unref all non-virgl objects；
3. flush object/control/cursor/config works；
4. wait control queue max slots；
5. wait cursor queue max slots；
6. `virtio_reset_device()`；
7. delete vqs。

### 13.2 Restore

`virtgpu_restore()`：

- find vqs；
- device ready；
- restore non-virgl objects；
- mode config resume。

### 13.3 VirGL restriction

KMD PM notifier 对 S4/hibernation 与 VirGL 有显式限制。不要从 non-virgl
hibernation restore 推断 VirGL/Venus/GFXStream 完整状态可恢复。

## 14. Object restore list

non-virgl objects 保存 `params` 到 restore list。恢复时：

- imported dma-buf resubmit；
- blob create；
- normal resource create；
- attach backing；
- 恢复 display objects。

该机制是 driver/Host resource namespace 重建，不是 Host GPU context 的完整
snapshot。

## 15. Failure matrix

| 症状 | KMD 检查 |
| --- | --- |
| probe 失败 | features/vqs/vbuf/modeset |
| no capsets | config/GET_CAPSET |
| context init fail | params/ring mask/context command |
| execbuffer ENOSYS | `has_virgl_3d` |
| queue wait timeout | descriptor starvation/device hang |
| response mismatch | response size/error header |
| fence timeout | emit/response/dequeue/fence context |
| old backing data | DMA sync/transfer fence |
| object free hang | Host unref callback/resource ref |
| hibernation fail | freeze/queue/restore list/VirGL restriction |

## 16. Recommended source order

1. `virtgpu_drv.c` probe/PM.
2. `virtgpu_kms.c` init/features/vqs.
3. `virtgpu_vq.c` vbuf/queue/response.
4. `virtgpu_object.c` GEM/resource/blob.
5. `virtgpu_submit.c` context/execbuffer.
6. `virtgpu_fence.c` timeline.
7. `virtgpu_ioctl.c` UAPI/syncobj.
8. `virtgpu_plane.c` display consumers.
9. `virtgpu_prime.c` dma-buf/restore.
10. QEMU device implementation.

## 17. Questions

1. 为什么 virtio KMD 不解析 Vulkan/GL command？
2. control/cursor queue 为什么分离？
3. vbuf 在 in-flight 期间保护什么？
4. fence seqno 为什么在 descriptor slot 确认后 emit？
5. 为什么 DMA sync 必须早于任何同 context fence signal？
6. Resource ID、GEM handle、GPU VA 和 dma-buf fd 有何不同？
7. `sg_dma_address()` 与 `sg_phys()` mem entry 如何选择？
8. object free 为什么要等 Host unref response？
9. `ring_idx` 如何形成独立 fence context？
10. 为什么 submit notify 不等于 GPU completion？
11. non-virgl object restore list 能/不能恢复什么？
12. 为什么 hibernation 限制不能外推到 live migration？
