# GFXStream 源码跟踪

## 1. 阅读范围

对应宏观文档：[gfxstream-overview.md](gfxstream-overview.md)。ASG ring
仍建立在共用 Guest KMD 的 context/ring/blob 和 execbuffer 之上；公共
transport、descriptor 回收和 fence completion 见
[shared-kmd-source-trace.md](shared-kmd-source-trace.md)。

本文跟踪 Guest GFXStream Vulkan ICD 如何建立 ASG ring、编码 Vulkan 命令、
通过 virtio-gpu 发送到 QEMU Rutabaga，并最终进入 Host
`libgfxstream_backend`。Rutabaga 自身代码另见
[rutabaga-source-trace.md](rutabaga-source-trace.md)。

宏观问题与本文源码入口的对应关系如下：

| 宏观问题 | GFXStream 的源码答案 | 本文位置 |
| --- | --- | --- |
| Guest UMD 做什么 | GFXStream Vulkan ICD、`ResourceTracker` 和 `VkEncoder` 序列化 API | 第 5–6 节 |
| Guest KMD 做什么 | 建 virtio context/ASG blob，运输 command/ring 并完成 fence | 第 3–4 节、共用 KMD |
| 如何利用 Host GPU | QEMU Rutabaga 经 FFI 调 backend，再由 Host Vulkan 执行 | 第 7–9 节 |
| 虚拟化形式 | API streaming；Guest ICD 与 Host decoder 通过有版本协议的 byte stream 通信 | 第 2、5–9 节 |
| 数据如何传输 | ASG ring blob 承载 Vulkan bytes，控制命令走 virtqueue，资源可用 dma-buf | 第 4、6–8 节 |
| 控制逻辑 | Device/capset 初始化、ASG context、queue flush 和 backend submit 分层 | 第 3–9 节 |
| 内存与所有权 | ResourceTracker 映射 Guest handle；backend 保有自己的 Host object | 第 6 节 |
| 同步与完成 | GFXStream seqno/fence 经 Rutabaga callback 和 QEMU AIO 返回 | 第 8、10 节 |
| 呈现 | external image/dma-buf、modifier 和 compositor WSI 决定可见性 | 第 11 节 |

## 2. 总调用链

```text
Guest Vulkan application
  -> GFXStream Vulkan ICD
  -> ResourceTracker + VkEncoder
  -> GFXStream command stream
  -> createVirtioGpuAddressSpaceStream()
  -> ASG ring in mappable blob
  -> Guest virtio-gpu
  -> control virtqueue
  -> QEMU rutabaga_cmd_submit_3d()
  -> rutabaga_submit_command()
  -> Rust RutabagaContext::submit_cmd()
  -> GfxstreamContext::submit_cmd()
  -> stream_renderer_submit_cmd()
  -> libgfxstream_backend
  -> Host Vulkan
```

## 3. Guest 设备初始化

### 3.1 选择 virtio-gpu render node

[`DrmVirtGpuDevice::openDevice()`](../../mesa/src/gfxstream/guest/platform/drm/DrmVirtGpuDevice.cpp:45)
枚举 DRM devices，匹配：

```text
PCI vendor 0x1af4
PCI device 0x1050
DRM_NODE_RENDER
driver version name == virtio_gpu
```

这证明 GFXStream ICD 仍把 Guest virtio-gpu 当作设备，而不是直接打开物理
GPU render node。

### 3.2 查询 capability

[`DrmVirtGpuDevice::init()`](../../mesa/src/gfxstream/guest/platform/drm/DrmVirtGpuDevice.cpp:147)
查询：

- 3D feature；
- capset query fix；
- resource blob；
- host-visible；
- cross-device；
- context init；
- supported capset IDs；
- debug name；
- downstream GFXStream 的 fence passing/guest-handle 参数。

然后 `GET_CAPS` 读取所选 GFXStream capset。Linux 非 Android 路径要求
`GFXSTREAM_VULKAN` 在 supported capset bitmask 中。

### 3.3 Context init

GFXStream 的 context 参数通常包括：

```text
CAPSET_ID = GFXSTREAM_VULKAN
NUM_RINGS = 2
optional DEBUG_NAME
```

随后 `DRM_IOCTL_VIRTGPU_CONTEXT_INIT` 建立 Guest virtio context。

## 4. ASG Ring 建立

### 4.1 从 capset 取 ring 参数

[`GetRingParamsFromCapset()`](../../mesa/src/gfxstream/guest/GoldfishAddressSpace/VirtioGpuAddressSpaceStream.cpp:13)
按 capset 读取：

- `ringSize`
- `bufferSize`
- `blobAlignment`

Guest page size 作为 alignment 下限。

### 4.2 创建 ring blob

[`createVirtioGpuAddressSpaceStream()`](../../mesa/src/gfxstream/guest/GoldfishAddressSpace/VirtioGpuAddressSpaceStream.cpp:68)
要求至少一个条件：

- `VIRTGPU_PARAM_HOST_VISIBLE`
- `VIRTGPU_PARAM_CREATE_GUEST_HANDLE`

然后创建：

```c
blob_mem   = VIRTGPU_BLOB_MEM_HOST3D
blob_flags = VIRTGPU_BLOB_FLAG_USE_MAPPABLE
blob_id    = 0
size       = align(ringSize + bufferSize, blobAlignment)
```

### 4.3 在 blob 上创建 context

Guest 将 `GFXSTREAM_CONTEXT_CREATE` 作为 execbuffer command 发出，并把刚创建
的 blob 作为 resource 引用。成功后才 `blob->wait()` 并 `mmap`。

映射布局：

```text
blob start -> struct asg_ring_storage
bufferPtr  -> ring command buffer
```

`asg_context_create()` 初始化 guest write position、host consumed position 和
transfer mode，再由 `AddressSpaceStream` 接管后续 API bytes。

## 5. Guest Vulkan ICD

### 5.1 Device 初始化

GFXStream Vulkan device 的源码位于：

- `mesa/src/gfxstream/guest/vulkan/gfxstream_vk_device.cpp`
- `mesa/src/gfxstream/guest/vulkan/gfxstream_vk_cmd.cpp`
- `mesa/src/gfxstream/guest/vulkan_enc/`

初始化过程中，`ResourceTracker::setupCaps()` 和 `setupFeatures()` 根据 Host
支持情况暴露或屏蔽 API capability。`GfxStreamConnectionManager` 获取
`VkEncoder`，建立 Guest-to-Host API byte stream。

### 5.2 API entrypoint

Guest 应用调用标准 Vulkan API。GFXStream entrypoints：

1. 创建 Guest wrapper object。
2. 分配 Guest object ID。
3. 用 `VkEncoder` 序列化调用。
4. 将命令写入 `VulkanStreamGuest`/address-space stream。
5. 对特殊资源调用 `ResourceTracker` 做 handle、memory、image 和 external
   memory virtualization。

GFXStream 不是把 Host Vulkan function pointer 暴露给 Guest 应用。

### 5.3 Command buffer

`vkBeginCommandBuffer`/`vkCmd*` 主要在 Guest 侧生成 staging command stream。
真正的 queue submit 通过 `ResourceTracker::on_vkQueueSubmit()` 处理，最终
把 command bytes flush 到 ASG ring。

`vkQueueFlushCommandsGOOGLE_encode_impl.cpp.inl` 展示三个关键点：

- 分配 GFXStream sequence number；
- 编码 `vkQueueFlushCommandsGOOGLE`；
- `stream->flush()` 让 ring 看到完整 command。

## 6. ResourceTracker 与资源类型

`ResourceTracker` 是 GFXStream Guest 的中央对象表，主要处理：

- VkDeviceMemory 的 guest allocation/import；
- VkImage 的 format、tiling、DRM modifier 和 memory requirements；
- VkBuffer 的 blob/backing；
- descriptor set 的 image/buffer handle；
- native buffer 和 Android hardware buffer；
- queue submit 与 fence/seqno。

典型调用：

```text
on_vkCreateImage()
  -> 记录 Guest image metadata
  -> 申请 memory requirements

on_vkBindImageMemory()
  -> 绑定 Guest memory/resource
  -> 编码 Host bind

on_vkQueueSubmit()
  -> flush command streams
  -> 关联 Host fence/seqno
```

格式映射不是一维字符串转换。`ResourceTracker` 还要判断 Host VirtGpu format、
external handle、modifier 和 image usage。某些 surface format 能被枚举，
不代表能作为 color attachment 或可导出 dma-buf。

## 7. QEMU 接收

QEMU 的 `VIRTIO_GPU_CMD_SUBMIT_3D` 进入
[`rutabaga_cmd_submit_3d()`](../../qemu/hw/display/virtio-gpu-rutabaga.c:348)：

1. 检查 `cs.size <= VIRTIO_GPU_MAX_CMD_SUBMIT_SIZE`。
2. `g_try_new0()` 分配临时 buffer。
3. `iov_to_buf()` 从 Guest SG 复制 command payload。
4. 校验实际 bytes。
5. 填 `struct rutabaga_command` 的 `ctx_id/cmd/cmd_size`。
6. 调 `rutabaga_submit_command()`。

GFXStream 的 Vulkan command 通常在 ASG ring blob 内，这里的 payload 仍可能
只是推进 ring 或 control message；实际大小取决于 GFXStream 协议路径。

## 8. Rutabaga 到 GFXStream

FFI 的 `rutabaga_submit_command()`
（`rutabaga_gfx_ffi-0.1.3/src/lib.rs:589`）：

1. `catch_unwind` 包住 Rust call。
2. 把 C `cmd/cmd_size` 转成 Rust mutable slice。
3. 把 input fence IDs 转成 slice。
4. 调 Rust `Rutabaga::submit_command()`。

Rust core 按 `ctx_id` 找 `RutabagaContext`，调用
`GfxstreamContext::submit_cmd()`
（`rutabaga_gfx-0.1.3/src/gfxstream.rs:199`）：

1. GFXStream component 在当前版本拒绝非空 input fences。
2. 校验 command bytes 4-byte 对齐。
3. 填 `stream_renderer_command`。
4. 调动态库符号 `stream_renderer_submit_cmd()`。

## 9. Host GFXStream decoder

固定 GFXStream commit `0ab696da6e393009c2ebeeac8f33d1c9531d940f` 中，
Host 关键路径：

```text
host/virtio_gpu_gfxstream_renderer.cpp
  -> host/renderer_impl.cpp
  -> host/vulkan/vk_decoder.cpp
  -> host/vulkan/vk_sub_decoder.cpp
  -> Host Vulkan dispatch
```

Host decoder 按 command opcode 解 Guest API 参数，维护 Host boxed handles、
resource mapping、external memory、image layout、queue submit 和 display。
`vulkan_stream.cpp` 负责从 Guest 输入读 bytes。

GFXStream 的 Host object handle table 与 Guest object ID 分开。Guest 伪造
一个数字 handle 不能直接变成 Host pointer。

## 10. Fence/completion

GFXStream Rust component 注册 `write_context_fence` callback。Host backend
完成 fence 时：

```text
stream_renderer fence callback
  -> write_context_fence()
  -> RutabagaFenceHandler
  -> QEMU virtio_gpu_rutabaga_fence_cb()
  -> schedule AIO bottom half
  -> virtio_gpu_rutabaga_aio_cb()
  -> match ctx_id/ring_idx/fence_id
  -> virtio response
```

QEMU callback 可能来自非 QEMU 主线程。源码明确使用
`aio_bh_schedule_oneshot()` 把 response 回到 QEMU AIO context，避免直接从此
callback thread调用 QEMU RCU/locking 路径。

## 11. Native WSI

GFXStream 常见 WSI 数据面：

1. Guest ICD 创建 external/dma-buf image。
2. GFXStream resource 映射到 Host Vulkan memory。
3. Guest compositor 导入 dma-buf。
4. compositor 采样/合成。
5. acquire/release fence 协调双方访问。
6. scanout 或 Host compositor 呈现。

源码审查点：

- `ResourceTracker` 的 format/modifier 映射；
- `on_vkCreateImage()` usage flags；
- external memory fd 类型；
- Host `external_memory.cpp` import/export；
- queue submit 中 image layout barrier；
- Guest compositor DRM modifier 支持。

现有 native WSI 诊断表明：swapchain 可创建、acquire-only 可稳定，首次把
native image 作为 color attachment 仍可能非法。这是 usage/backing 问题，
不是简单“Rutabaga submit 不通”。

## 12. 失败点

| 症状 | 源码检查 |
| --- | --- |
| ICD 不出现 | supported capset bitmask、ICD JSON |
| context create 失败 | `DrmVirtGpuDevice::init()`、QEMU capset |
| ring 创建失败 | blob flags、alignment、host-visible |
| queue submit 无效果 | command flush/seqno、Rutabaga submit |
| fence timeout | GFXStream fence callback、QEMU BH、ring idx |
| image 内容错误 | ResourceTracker format/layout/memory |
| swapchain 黑屏 | external memory、modifier、compositor WSI |

## 13. 推荐源码阅读顺序

1. `DrmVirtGpuDevice.cpp`：Guest device/capset。
2. `VirtioGpuAddressSpaceStream.cpp`：ASG blob 和 ring。
3. `gfxstream_vk_device.cpp`：ICD 初始化。
4. `ResourceTracker.cpp`：对象、格式和 memory。
5. `gfxstream_vk_cmd.cpp`：command buffer。
6. `vkQueueFlushCommandsGOOGLE_encode_impl.cpp.inl`：flush/seqno。
7. QEMU `virtio-gpu-rutabaga.c`：transport 和 fence。
8. Rutabaga `gfxstream.rs`：component FFI。
9. Host `vk_decoder.cpp`、`external_memory.cpp`、`display_vk.cpp`。

## 14. 检查题

1. GFXStream Guest ICD 如何发现 virtio-gpu render node？
2. ASG blob 中 ring storage 和 buffer 的边界在哪里？
3. GFXStream context create 为什么先于 mmap ring？
4. `ResourceTracker` 为什么要覆盖 image create、bind 和 queue submit？
5. QEMU 的 `SUBMIT_3D` 与 ASG ring 哪一部分是控制面？
6. FFI 如何把 QEMU C command 转成 Rust slice？
7. GFXStream component 当前为什么拒绝非空 input fences？
8. fence callback 为什么不能直接在主线程之外回复 QEMU？
9. native image 作为 color attachment 需要哪些 usage/format/backing 条件？
10. `wl_shm` WSI 与 native dma-buf WSI 在源码中如何分叉？
