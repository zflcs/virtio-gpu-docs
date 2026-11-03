# Virtio-GPU ROCm Compute 源码跟踪

## 1. 文档状态

对应专题：[Userptr 与 ROCm compute](userptr-rocm-compute.md)。

本文跟踪 AMD 的 **尚未合入** ROCm native context MR
[`!1568`](https://gitlab.freedesktop.org/virgl/virglrenderer/-/merge_requests/1568)。
固定分析快照：

```text
MR state: opened
head commit: e6588b4e98045c76baab0e40682f43ac0913caaa
last observed: 2026-10-29
```

它不是当前 Linux/Mesa/QEMU 的稳定 ABI，不能把其中的常量、命令或源码行为
写成“已支持”。本文用于研究方向、审查设计和规划未来版本。

## 2. 完整目标路径

```text
Guest ROCm application
  -> HIP / ROCr
  -> Guest libhsakmt virtio frontend
  -> vDRM ccmd batch
  -> DRM_IOCTL_VIRTGPU_EXECBUFFER
  -> Guest virtio-gpu KMD
  -> QEMU
  -> virglrenderer ROCm context
  -> Host libhsakmt
  -> /dev/kfd + amdgpu
  -> GPU compute/copy queues
```

Rutabaga 不在此提案的数据路径中；ROCm backend 挂在 virglrenderer 的 native
context 框架下。

## 3. 为什么复用 vDRM 框架

`hsakmt_context.h` 将：

```c
#define vhsakmt_base_context drm_context
#define vhsakmt_base_object  drm_object
#define vhsakmt_ccmd        drm_ccmd
```

因此 ROCm backend 复用了 `drm_context` 的：

- `vdrm_ccmd_req/rsp` 请求响应格式；
- command 长度、alignment 和 response bounds 检查；
- resource table；
- blob table；
- context 通用 destroy/attach/get_fd 回调；
- fence retire 接口。

ROCm 在此基础上增加 object type、HSA node、VA manager 和 libhsakmt 操作。

这是一种“底层驱动 UAPI/服务 API proxy”模型，不是 PCI passthrough，也不是
Vulkan compute remoting。

## 4. Host backend 初始化

`vhsakmt_device_init()`：

1. `hsaKmtOpenKFD()` 打开 Host KFD。
2. `hsaKmtGetVersion()` 填 capset version。
3. `hsaKmtAcquireSystemProperties()`。
4. `hsaKmtGetNodeProperties()` 枚举 node/GPU。
5. 根据 Host system RAM、VRAM 和 GPU 数计算 VA 保留区。
6. 可通过环境变量覆盖：
   - `VHSA_VAMGR_VM_TYPE`
   - `VHSA_VAMGR_VM_BASE_ADDR`
   - `VHSA_VAMGR_VM_KFD_SIZE`
   - `VHSA_VAMGR_VM_SCRATCH_SIZE`
   - `VHSA_DUMP_VA`
7. `mmap(PROT_NONE)` 保留 Host VA。

Backend 还建立：

- normal KFD VA region；
- 每 GPU scratch aperture；
- node/GPU 属性表；
- Host `hsaKmt*` 对象类型表。

## 5. Capset

`virgl_renderer_capset_hsakmt` 当前只包含：

```text
wire_format_version
KFD version major/minor/patchlevel
context_type
```

与 Venus 的丰富 capset 相比，这个 ABI 很早期。Guest 需要知道：

- Host renderer 是否编译 ROCm backend；
- Host libhsakmt/KFD 版本；
- context type 是否为 AMDGPU；
- 后续通过 query commands 获取 node/GPU/memory/link 属性。

MR 当前常量仍叫 `VIRGL_RENDERER_CAPSET_HSAKMT = 8`，而 Linux userptr v9
已把 proposal capset 改为 `ROCM = 10`。这是全栈 ABI 错位之一。

## 6. Context 和 object table

`vhsakmt_device_create()`：

1. 分配 `vhsakmt_context`。
2. `drm_context_init()` 建立 resource/blob hash table。
3. 注册回调：
   - `destroy`
   - `attach_resource`
   - `get_blob`
   - `submit_fence`
   - `detach_resource`
   - `retire_fences`
   - `free_object`
   - `export_opaque_handle`
4. 保存 debug name 和 backend state。

Object type：

| Type | Host object |
| --- | --- |
| `HOST_MEM` | hsaKmt allocation |
| `USERPTR` | 连续 Host VA/注册的 Guest backing |
| `EVENT` | `HsaEvent *` |
| `QUEUE` | `vHsaQueueResource *` |
| `DOORBELL_PTR` | Host doorbell aperture alias |
| `DOORBELL_RW_PTR` | queue rptr/wptr page |
| `QUEUE_MEM` | ring/AQL queue memory |
| `DMA_BUF` | exported dma-buf fd |
| `SCRATCH_MAP_MEM` | per-GPU scratch range |

对象同时通过 `res_id` 和 `blob_id` 索引。释放时按 object type 调不同 Host
destroy API。

## 7. Blob 与 userptr

`vhsakmt_device_get_blob()`：

### 7.1 control shmem

`blob_id == 0` 调 `drm_context_get_shmem_blob()`，建立 command
request/response shmem。

### 7.2 userptr with explicit VA

如果 blob flag 是 `USE_USERPTR` 且 `blob->u.va_handle` 非空：

- 创建 `VHSAKMT_OBJ_USERPTR`；
- 保存连续 VA；
- Host libhsakmt 后续 register/map 该 VA。

### 7.3 userptr from iovecs

如果只有 iovec：

1. 创建 userptr object。
2. 保存 iov/iov_count。
3. `mmap()` 一块匿名连续 VA。
4. 稍后在 register 命令中用 `mremap(..., MREMAP_MAYMOVE |
   MREMAP_FIXED | MREMAP_DONTUNMAP, target)` 把每个 iovec 映射到连续目标。
5. `hsaKmtRegisterMemoryWithFlags()` 注册连续 VA。

这条路径不直接使用 `blob->u.va_handle`；从 QEMU/Rutabaga 可以看到 iovec 与
VA 两种表示并不完全统一。

### 7.4 resource identity

成功 get_blob 后：

```text
blob.type = VIRGL_RESOURCE_VA_HANDLE
blob.u.va_handle = obj->bo
```

virglrenderer 的 VA resource 不再二次 map，直接把 Host VA 当 resource
identity。这减少了二次映射，但也意味着 Host VA 生命周期必须与 Guest
resource、Host hsaKmt object 和 queue/event 引用严格一致。

### 7.5 current map_info bug

当前 `vhsakmt_device_get_blob()` 没有设置 `blob->map_info`。公开评审在
2026-10-29 指出，后续 `virgl_renderer_resource_get_map_info()` 会失败。

此外，iov 路径创建匿名连续映射、mremap 原始 iov、再 register 的复杂度高，
需要验证：

- iov 页是否和 Guest 页保持同一 backing；
- `MREMAP_DONTUNMAP` 后源映射是否保留；
- 失败时每个 iov 如何恢复；
- Host hsaKmt deregister 与 munmap 顺序；
- DMA/cache coherence。

## 8. Memory commands

`vhsakmt_ccmd_memory()` 的 switch 包含：

- ALLOC/FREE；
- MAP_TO_GPU_NODES；
- MAP_MEM_TO_GPU；
- REG_MEM_WITH_FLAG；
- DEREG_MEM；
- MAP_USERPTR；
- EXPORT_DMABUF；
- AMDGPU import/export/VA/BO free；
- share memory；
- set memory policy；
- SVM get/set attrs；
- BO query/metadata。

### 8.1 Allocation

`vhsakmt_alloc_memory()` 按 flag 分叉：

- scratch -> 从 per-node scratch VA manager 分配；
- `NoAddress` -> `hsaKmtAllocMemory()` 无地址对象；
- host/GTT/only-address -> host path；
- device VRAM -> device path。

Host 路径先分配 VA manager fragment，`PROT_NONE` 保留空间，再
`hsaKmtAllocMemory()`，并要求返回地址与预期 Host VA 一致。失败时释放
HSA object 和 VA fragment。

### 8.2 Register userptr

`REG_MEM_WITH_FLAG`：

- 如果有 iov，使用 `VHSA_REGISTER_RANGES` 或 mremap 合并连续 VA；
- 调 `hsaKmtRegisterMemoryWithFlags()`；
- 成功后将 object 的 bo/size 更新为连续 VA。

如果只是普通 VA，直接注册该 Host VA。

### 8.3 DMA-BUF

`EXPORT_DMABUF` 调用 `hsaKmtExportDMABufHandle()`，保存 fd/offset。
`DEREG_MEM` 可能同时清除 DMA-BUF object。

Host export 的 fd 如何安全转发给 Guest、由谁 close、是否经过 QEMU fd
转递，需要 QEMU/QMP/virtio 协议配合；不能仅凭 renderer source 认为 fd
已跨 VM。

## 9. Queue 路径

`vhsakmt_queue_create()`：

1. 按 NodeId 找 Host node。
2. 分配 `vHsaQueueResource`。
3. 调 `hsaKmtCreateQueue()` 或 `hsaKmtCreateQueueExt()`。
4. 要求 Host 返回 non-NULL doorbell。
5. 记录 node doorbell base 和 queue doorbell offset。
6. 按需创建 doorbell blob。
7. 非 AQL queue 创建 rptr/wptr blob。
8. 创建 `VHSAKMT_OBJ_QUEUE`。
9. 将 queue mem/rw mem object 与 queue object相互绑定。

### 9.1 Doorbell

Guest 不能直接使用 Host BAR doorbell pointer。Renderer 暴露：

- `host_doorbell` / `host_doorbell_offset`；
- doorbell blob；
- rw pointer blob。

Guest frontend需要把这些 Host 语义转换成 Guest 的 proxy doorbell 或 control
command。具体 Guest libhsakmt ABI不在本 MR 仓库中，必须对照 ROCm 前端补丁。

### 9.2 Queue destroy

`vhsakmt_free_queue_obj()`：

1. `hsaKmtDestroyQueue()`。
2. 清 queue/rw_mem/queue_mem 双向指针。
3. 释放关联 memory object。
4. 释放 queue structure。

Queue memory 在 queue 仍引用时返回 `-EBUSY`，要等 Guest detach/queue destroy。

### 9.3 Update/CU mask/GWS

支持 update queue、set CU mask、alloc queue GWS。它们直接映射到 Host
`hsaKmt*` API。AI QoS 可能依赖这些控制，但当前 proposal 没有完整的
multi-tenant quota、preemption 和 fair-share policy。

## 10. Event 路径

`vhsakmt_ccmd_event()`：

- CREATE：Host `hsaKmtCreateEvent()`，返回 copied `HsaEvent` 和 Host handle；
- DESTROY；
- SET；
- RESET；
- QUERY_STATE；
- SET_TRAP；
- `WAIT_ON_MULTI_EVENTS` 当前明确返回 `-ENOSYS`。

Event object 保存 Host `HsaEvent *`。Host handle 被复制进 response 的
`vHsaEvent`；Guest 前端如何使用该 handle 需要严格验证。不能把 Host pointer
直接暴露给恶意 Guest。

当前缺少 multi-event wait 是 AI runtime 兼容性的明显风险，因为 ROCm/HIP
可能等待多个队列/事件依赖。

## 11. Fence/completion

当前 `vhsakmt_device_submit_fence()`：

- 只对 `ring_idx == 0` 记录 `last_fence_id` 并 retire；
- 其他 ring 直接返回 0，没有在可见源码中建立 Host GPU fence 到 virtio
  ring fence 的映射。

`vhsakmt_context_retire_fences()` 也只 retire ring 0 的 last fence。

这说明 proposal 的 Host compute completion 更可能依赖 HSA event/queue 内部
API，而 virglrenderer 的通用 ring fence 支持尚不完整。必须明确：

- HSA event 完成；
- queue kernel 完成；
- VIRTIO fence response；
- Guest sync fd；
- 多 ring timeline；

不能将“queue command handled”当作“kernel execution complete”。

## 12. Host GPU VM / VA 协商

Backend 支持两种 VA 模式：

- fixed base：选择固定 Host VA 区间；
- negotiated：读取 Guest VM start，尝试对齐/保留一个双方都可用的区间。

VA 大小按 `(system RAM + VRAM) * 2` 估计，上限为
`VHSA_VAMGR_VM_MAX_KFD_SIZE`，再加 per-GPU scratch range。

这是整个提案最敏感的部分之一：

- Host `mmap(PROT_NONE)` 只是 VA reservation；
- GPU page table 仍由 Host amdgpu/KFD 管理；
- Guest 看到的 VA 与 Host GPU VA 可能通过固定/协商布局对应；
- `mremap` 可以改变 Host VA，却不应改变 Guest visible object identity；
- process teardown/reset 必须同时撤销 GPU PTE、DMA map 和 Host VA。

## 13. 当前源码审查清单

### ABI

- MR capset 8 vs Linux ROCm capset 10；
- `VIRGL_RENDERER_USE_ROCM` 与 `USE_GUEST_VRAM` 都使用 bit 14；
- QEMU RFC 与 Linux v9 feature/blob flag 编号不一致。

### Memory

- `blob->map_info` 未初始化；
- iovec -> contiguous VA 的 mremap 生命周期；
- userptr register/unregister 与 pin/DMA coherence；
- Host VA fixed/negotiated 冲突；
- dma-buf fd 所有权；
- VRAM quota 和 eviction。

### Queue

- Host doorbell pointer 不应直接给 Guest；
- proxy doorbell 与 aggregated doorbell；
- queue memory/refcount；
- queue update 和 CU mask/GWS；
- preemption/reset。

### Event/fence

- multi-event wait `-ENOSYS`；
- Host event handle身份；
- ring index/timeline；
- kernel completion 与 virtio fence 的映射；
- device lost 和 queue reset。

### Lifecycle

- context deinit 与 backend global VA/scratch 生命周期；
- queue/memory/event 循环引用；
- `-EBUSY` 重试；
- Guest unref/Host KFD free 的顺序；
- hibernation/migration/reset。

## 14. 与其他路径的区别

| 维度 | Venus | AMD native DRM | ROCm proposal |
| --- | --- | --- | --- |
| Guest API | Vulkan | DRM/amdgpu UAPI | HSAKMT/ROCm |
| Host API | Vulkan | libdrm_amdgpu | libhsakmt |
| Kernel fd | render node | render node | `/dev/kfd` + amdgpu |
| Main object | Vk object | GEM/VM/CS | BO/queue/event/SVM |
| AI stack | Vulkan compute | ROCm 不完整 | 面向 ROCm/HIP |
| Status | 已实现 | 多个 backend 上游 | 未合入 |

它最接近“ROCm compute native context”，但仍由 virglrenderer 软件代理，
不能获得 VFIO/SR-IOV/MIG 的硬件隔离属性。

## 15. 推荐验证顺序

未来实验解禁后，按依赖顺序：

1. Host libhsakmt 能独立打开 KFD、枚举 node/GPU。
2. virglrenderer ROCm backend init/capset。
3. Guest context create。
4. control shmem blob。
5. AQL queue + doorbell + event。
6. host/device memory alloc 和 map。
7. userptr register/map。
8. small HIP kernel + event wait。
9. fence/timeline 和 multi-event。
10. GEMM 与 decode 小 kernel。
11. multi-GPU peer/collective。
12. reset/OOM/teardown。

当前只记录计划，不执行。

## 16. 检查题

1. 为什么 ROCm backend 复用 `drm_context` 而不复用 `vrend`？
2. `VIRGL_RESOURCE_VA_HANDLE` 与普通 dma-buf/fd resource 有什么区别？
3. Host VA manager 的 fragment 分配为什么不能替代 GPU page table？
4. iovec->mremap->hsaKmtRegister 路径是否真正 zero-copy？
5. Guest userptr pin 与 Host hsaKmt userptr register 如何对应和释放？
6. Host event handle 能否安全返回 Guest？
7. multi-event wait 缺失会破坏哪些 HIP/ROCr 场景？
8. 当前 fence 实现为什么不足以证明完整 GPU timeline？
9. queue memory 的 `-EBUSY` 生命周期说明了什么？
10. 为什么该 proposal 仍不是 VFIO/SR-IOV/MIG？
