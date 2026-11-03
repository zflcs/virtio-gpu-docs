# DRM GEM 与 PRIME 源码跟踪

## 1. 目标

对应基础知识：

- [GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)
- [DMA-BUF、DMA-FENCE、Reservation 与 Sync File](dma-buf-fence-source-trace.md)
- [Virtio-GPU 共用 Linux KMD 源码跟踪](../virtio-gpu/shared-kmd-source-trace.md)

本文回答几个经常被混用的对象：

```text
driver-private BO
GEM object
GEM handle
fake mmap offset
dma-buf object/fd
GPU VA
```

它们是不同 namespace 或生命周期对象，不是同一个 ID 的不同名字。

固定源码：

```text
Linux 6b4b543fa18757297ac756a6fe6e64f506b7844a
drivers/gpu/drm/drm_gem.c
drivers/gpu/drm/drm_prime.c
drivers/gpu/drm/drm_gem_shmem_helper.c
include/drm/drm_gem.h
include/uapi/drm/drm.h
```

本文不运行 GEM/PRIME、VM 或 GPU workload 实验。

## 2. 一张图

```text
driver allocation
  -> struct driver_bo containing drm_gem_object
  -> drm_gem_handle_create(file)
       -> u32 handle in that DRM file
       -> handle reference + object ref
  -> drm_gem_create_mmap_offset()
       -> fake offset in device mmap namespace
  -> DRM_IOCTL_PRIME_HANDLE_TO_FD
       -> dma_buf object
       -> fd in caller
  -> DRM_IOCTL_PRIME_FD_TO_HANDLE in another file/device
       -> driver import/attach/map
       -> new local GEM handle
```

## 3. GEM Object 与 driver BO

DRM 的 `drm_gem_object` 通常嵌入 driver BO：

```text
struct amdgpu_bo
  -> struct ttm_buffer_object
       -> struct drm_gem_object?

struct virtio_gpu_object
  -> struct drm_gem_shmem_object
       -> struct drm_gem_object
```

具体嵌入方式依 driver。公共 GEM object 保存：

- device；
- size；
- refcount；
- handle count；
- dma_resv；
- VMA node/fake offset；
- optional shmem file；
- exported/imported dma-buf pointers；
- object functions。

driver-private fields 保存 backing placement、GPU VA、resource ID 等。

## 4. Object 初始化

### 4.1 Shmem-backed

[`drm_gem_object_init()`](../../linux/drivers/gpu/drm/drm_gem.c:184)：

- 调用 common private init；
- 创建 shmem file；
- 存在 huge mount 时使用 huge shmem mount。

### 4.2 Private backing

[`drm_gem_private_object_init()`](../../linux/drivers/gpu/drm/drm_gem.c:219)
初始化：

- kref；
- size；
- `dma_resv_init()`；
- GPUVA lock/context；
- VMA node；
- LRU link。

private object 无公共 shmem backing，由 driver 负责实际 memory。

## 5. Handle 是 DRM file namespace ID

### 5.1 Create

[`drm_gem_handle_create()`](../../linux/drivers/gpu/drm/drm_gem.c:554)
调用 create_tail：

1. 增加 handle-count reference；
2. 在 `drm_file.object_idr` 分配 u32；
3. `drm_vma_node_allow()` 授权该 DRM file 映射；
4. 可选 driver `open` callback；
5. 把 object 发布到 handle table。

调用此函数时 object 必须已完全初始化。

### 5.2 Lookup

[`drm_gem_object_lookup()`](../../linux/drivers/gpu/drm/drm_gem.c:885)
只查当前 `drm_file`：

```text
fileA handle 7
  and
fileB handle 7
```

通常指向不同 objects。不要把 handle 通过 socket 发给另一进程。

lookup 返回 object reference，调用者最终必须 `drm_gem_object_put()`。

### 5.3 Close

[`drm_gem_handle_delete()`](../../linux/drivers/gpu/drm/drm_gem.c:402)：

1. 从 file IDR 摘除 handle；
2. 调 driver close callback；
3. 从 PRIME handle/buf cache 移除；
4. 撤销该 file 的 VMA mapping 权限；
5. 减少 handle reference；
6. 最后一个 handle 移除 GEM name/exported dma-buf cache；
7. object refcount 归零后才释放 BO。

GEM_CLOSE 是“关闭这个 file 的 handle”，不等于立即释放所有共享 backing。

## 6. Fake mmap offset

GEM mmap 使用 fake offset：

```text
DRM_IOCTL_MODE_MAP_DUMB / CREATE_DUMB
  -> drm_gem_create_mmap_offset()
  -> fake offset
  -> mmap(drm_fd, offset, size)
  -> drm_gem_mmap()
```

[`drm_gem_create_mmap_offset()`](../../linux/drivers/gpu/drm/drm_gem.c:625)
在 `vma_offset_manager` 中建立 object range。

[`drm_gem_mmap()`](../../linux/drivers/gpu/drm/drm_gem.c:1379)：

1. 根据 `vm_pgoff` 查 object；
2. 拒绝正在销毁的 0-ref object；
3. 检查 `drm_vma_node_is_allowed()`；
4. 调 `drm_gem_mmap_obj()`；
5. 安装 object-specific `vm_ops`。

fake offset 不是 GPU VA，也不是 dma-buf fd 或 physical address。

### 6.1 VMA authorization

`drm_vma_node_allow()`/`revoke()`让 GEM object 的 mmap 权限绑定 DRM file。
只有持有相应 handle/file 权限的用户才能查 offset。

这是防止 userspace 通过猜 offset 映射另一个 client object 的一层保护。

## 7. Shmem GEM

[`drm_gem_shmem_create()`](../../linux/drivers/gpu/drm/drm_gem_shmem_helper.c:155)
创建 shmem-backed object。

[`drm_gem_shmem_create_with_handle()`](../../linux/drivers/gpu/drm/drm_gem_shmem_helper.c:490)
再创建 userspace handle。

### 7.1 Page/SG

shmem pages 可以：

- mmap 给 CPU；
- vmap；
- pin；
- 通过 `drm_gem_shmem_get_pages_sgt()`生成 sg；
- export PRIME；
- import 到其他 device。

[`drm_gem_shmem_mmap()`](../../linux/drivers/gpu/drm/drm_gem_shmem_helper.c:752)
把 mmap 转到 shmem file。

### 7.2 与 virtio-gpu

virtio-gpu 普通 2D resource 使用 shmem GEM；Host QEMU 可通过 Guest backing
mem entries 访问对应 pages。

blob/Host3D 则可能由 Host renderer 分配，不再等同普通 Guest shmem GEM。

## 8. PRIME Export

[`drm_gem_prime_handle_to_fd()`](../../linux/drivers/gpu/drm/drm_prime.c:510)
把 local GEM handle 导出成 dma-buf fd。

核心 [`drm_gem_prime_handle_to_dmabuf()`](../../linux/drivers/gpu/drm/drm_prime.c:431)：

1. 校验 handle；
2. 查 file PRIME cache；
3. 如果 object 来自 import，返回原 dma-buf；
4. 如果已导出，复用 object cache；
5. 否则调用 driver object export 或 generic `drm_gem_prime_export()`；
6. 建立 handle<->dma-buf cache。

### 8.1 为什么缓存

同一 GEM object 多次 export 应保持稳定 dma-buf identity/cache，避免创建多个
独立 wrapper 导致不一致 reservation/lifetime。

### 8.2 Export 不等于 alloc/copy

通用 PRIME helper 把现有 GEM object 包装成 dma-buf；它不自动复制内容，也不
保证 arbitrary importer 能 DMA map。

## 9. PRIME Import

[`drm_gem_prime_fd_to_handle()`](../../linux/drivers/gpu/drm/drm_prime.c:292)：

1. `dma_buf_get(fd)`；
2. 查当前 file 是否已有这个 dma-buf 的 handle；
3. 有则复用 handle；
4. 否则调用 driver import 或 generic import；
5. 建立本地 GEM object；
6. `drm_gem_handle_create_tail()`；
7. 缓存 dma-buf -> handle。

### 9.1 Generic import

[`drm_gem_prime_import()`](../../linux/drivers/gpu/drm/drm_prime.c:1036)
调用 import_dev：

- 如果 dma-buf 来自同一 DRM device 的 GEM，则直接增加原 GEM ref；
- 否则 `dma_buf_attach()`；
- `dma_buf_map_attachment_unlocked()`取 sg；
- driver `gem_prime_import_sg_table()`建立本地 BO；
- 保存 `import_attach`；
- 共享 dma-buf reservation。

这揭示两种身份：

```text
same-device re-import
  -> original GEM object

cross-driver import
  -> new local GEM wrapper + original dma-buf attachment
```

## 10. dma-buf fd 与 GEM handle 不可互换

| 属性 | GEM handle | dma-buf fd |
| --- | --- | --- |
| namespace | 一个 DRM file | fd table/SCM_RIGHTS |
| 类型 | u32 | file descriptor |
| 跨进程 | 不能直接传 | 可以传 |
| 跨 driver | 不可用 | 标准接口 |
| 表示 | local object handle | shared dma-buf object |
| 内容复制 | 无 | 无（PRIME wrapper） |

mmap offset 是第三种 namespace：

| 属性 | fake mmap offset |
| --- | --- |
| namespace | DRM device vma_offset_manager |
| 用途 | 在该 DRM fd 上 mmap |
| 跨进程 | 不能替换 dma-buf fd |

## 11. Reservation/Fence

GEM private init 建立 dma_resv。PRIME import 时：

```text
obj->resv = dma_buf->resv
```

所以本地 importer 与 exporter共享同一个 reservation object，用于：

- implicit synchronization；
- CPU access fence wait；
- DMA map；
- migration/export。

explicit syncobj/sync_file 仍可以在 driver submit 中独立使用。

## 12. Virtio-GPU resource 映射

```text
Guest ioctl create resource
  -> virtio_gpu_object_create()
  -> drm_gem_object_init/shmem init
  -> virtio resource create command
  -> drm_gem_handle_create()
  -> userspace u32 handle
```

PRIME export：

```text
VIRTGPU resource/GEM
  -> dma-buf fd
  -> another Guest API/process
```

PRIME import：

```text
foreign dma-buf fd
  -> import_attach + local GEM
  -> virtio resource attach backing
  -> QEMU/renderer
```

resource ID、GEM handle 和 dma-buf fd 的身份不同：

- resource ID：virtio protocol object ID；
- GEM handle：Guest DRM file namespace；
- dma-buf fd：Guest process fd；
- GPU VA：GPU page table mapping。

## 13. 虚拟化边界

### 13.1 Guest/Host

Guest GEM handle 不会传到 Host。跨边界通常是：

- Guest resource ID；
- QEMU iovec/mem entry；
- dma-buf/opaque fd；
- UUID；
- shared blob mapping。

### 13.2 Host renderer object

Host Vulkan/DRM BO、dma-buf fd 和 Guest GEM 各自有生命周期。Host proxy 必须
建立显式映射并保活 backing，直到 Host queue 完成。

### 13.3 VM teardown

销毁顺序应至少覆盖：

```text
stop new submissions
  -> wait/cancel GPU work
  -> unmap context resources
  -> detach dma-buf/backing
  -> close handles/fds
  -> release GEM object
  -> release Host renderer object
```

## 14. AI workload 含义

- framework allocator 大块池化 GEM/dma-buf；
- 进程间共享 tensor 可能使用 PRIME/syncobj/exportable fd；
- 多 GPU peer buffer 需要 dma-buf import + P2P；
- KV cache/pinned tensor 的 handle 生命周期应跨请求；
- custom extension 可能直接操作 DRM/GEM handles；
- VM migration 需要重建 Guest GEM namespace，不能迁移 numeric handle；
- checkpoint 需要 drain GPU writes 并复制 backing 内容。

## 15. 失败矩阵

| 症状 | 优先检查 |
| --- | --- |
| handle lookup ENOENT | 错误 DRM file、handle 已 close、namespace |
| handle 在另一进程无效 | 应传 dma-buf/syncobj fd，而非 handle |
| mmap offset EACCES | VMA authorization/错误 fd |
| mmap range invalid | fake offset/size/lookup |
| export fd 失败 | export cache/object callback |
| fd import EINVAL | fd 不是 dma-buf/权限/格式 |
| import 成功但内容错 | backing/layout/modifier/cache |
| same-device 行为不同 | self-import 复用原 GEM 与 reservation |
| GEM_CLOSE 后仍占用内存 | 其他 handle/import/VMA/fence refs |
| Host queue 写完前释放 | renderer reference/fence lifetime |
| VM 恢复后 handle 错乱 | numeric handle 被错误保存 |
| 大模型内存泄漏 | dma-buf/attachment/VMA/context未释放 |

## 16. 审查清单

### 16.1 Object identity

- ID 是 GEM handle、resource ID、dma-buf fd、GPU VA 还是 mmap offset？
- 属于哪个 DRM file/进程/VM？
- 谁持有 object reference？

### 16.2 Memory

- shmem、VRAM、TTM、Host3D 还是 imported dma-buf？
- 是否 pin？
- backing 能否迁移？
- coherence 和 CPU access 如何 bracket？

### 16.3 Synchronization

- dma_resv 是否共享？
- 哪些 fence 用时？
- reset/teardown 如何 signal？

### 16.4 Virtualization

- Guest namespace 与 Host object 如何对应？
- 跨边界传 handle还是 fd/resource？
- destination VM 如何重建 object table？

## 17. 推荐源码顺序

1. `drm_gem_object_init/private_object_init`。
2. handle create/lookup/delete。
3. fake mmap offset 与 VMA authorization。
4. shmem create/pages/sg/mmap。
5. PRIME handle-to-fd/fd-to-handle。
6. PRIME import self-device 与 cross-driver 分支。
7. virtio `virtgpu_object.c` resource create/unref。
8. dma-buf/fence/syncobj 文档。

## 18. 检查题

1. GEM object、GEM handle 和 driver BO 的关系是什么？
2. 为什么两个进程可以同时有 handle 7，但指向不同 object？
3. GEM handle 为什么不能直接通过 socket 传？
4. fake mmap offset 属于什么 namespace？
5. DRM file 的 VMA authorization 防止什么？
6. PRIME export 是否会复制 buffer 内容？
7. 同设备 re-import 为什么可以复用原 GEM？
8. 跨 driver import 为什么需要 attachment/sg？
9. dma-buf fd、GEM handle 和 resource ID 的 ownership 差在哪？
10. GEM_CLOSE 后为什么对象可能仍存活？
11. PRIME import 后 dma_resv 为什么共享？
12. shmem GEM 与 Host3D blob 的区别是什么？
13. VM 迁移为什么不能保存 numeric GEM handle？
14. AI allocator 为什么倾向大块池化 GEM/dma-buf？
15. VM teardown 的 unmap/detach/close/release 顺序是什么？

## 19. 与其他专题

- [DMA-BUF、DMA-FENCE、Reservation 与 Sync File](dma-buf-fence-source-trace.md)
- [DRM Syncobj 与 Timeline 同步](drm-syncobj-source-trace.md)
- [GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)
- [Virtio-GPU 共用 Linux KMD 源码跟踪](../virtio-gpu/shared-kmd-source-trace.md)
- [Virtio-GPU Userptr 与 ROCm 社区动向](../virtio-gpu/userptr-rocm-compute.md)
