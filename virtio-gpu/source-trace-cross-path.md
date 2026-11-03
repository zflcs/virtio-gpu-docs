# Virtio-GPU 四路径源码横向对照

本文是四份逐路径源码跟踪文档的横向对照，不作为唯一源码入口。函数级主文档
分别是 `virgl-source-trace.md`、`venus-source-trace.md`、
`gfxstream-source-trace.md` 和 `rutabaga-source-trace.md`。本文用于比较
同一宏观问题在四条路径中的不同实现。

四条主路径共用 Guest Linux `virtio_gpu` KMD。probe、feature、virtqueue、
context、GEM/resource/blob、execbuffer、fence/syncobj 与 hibernation/reset
的公共函数级证据统一见
[shared-kmd-source-trace.md](shared-kmd-source-trace.md)。本文在路径差异
处引用该文档，避免把公共 transport 行为误认为某个 API remoting 后端特有
的实现。

`vhost-user-gpu` 是同一 Guest virtio-gpu ABI 的外置后端部署模式，不是新的
capset 或第五条 API 路径。QEMU、外置进程、virtqueue 和 display socket 的
边界见 [vhost-user-gpu-source-trace.md](vhost-user-gpu-source-trace.md)。

本文面向已经会写普通 GPU 程序、但还不熟悉虚拟化边界的读者。读完并完成
检查题后，应能独立回答：

- 一条 OpenGL、Vulkan、GFXStream 或 native DRM 命令在哪里编码、传输、
  解码和执行。
- Guest 看到的是 PCI 设备、DRM 设备、renderer 还是厂商 GPU。
- 每一块内存在哪里有副本，谁 pin、谁 map、谁负责同步、谁可以释放。
- 哪一层拥有 context、queue、fence 和 resource 生命周期。
- 渲染正确但呈现错误、fence 不返回、capset 不存在等故障分别应查哪里。

## 0. 宏观问题与源码章节映射

四份宏观文档统一回答“Guest UMD/KMD、Host GPU 使用、虚拟化形式、数据传输、
控制逻辑、内存、同步和呈现”。本文按同一组问题提供源码证据：

| 宏观问题 | VirGL 证据 | Venus 证据 | GFXStream 证据 | Rutabaga 证据 |
| --- | --- | --- | --- | --- |
| Guest UMD | Gallium driver / `virgl_encode.c` | `vn_renderer*` / `vn_ring.c` | GFXStream ICD / `VulkanStreamGuest` | Guest 使用所选 component 的 UMD |
| Guest KMD | `virtgpu_submit.c` / `virtgpu_vq.c` | 同一 virtio-gpu KMD + blob/context | 同一 KMD + ASG ring blob | 同一 KMD；Rutabaga 仅在 Host |
| Host GPU 使用 | vrend -> Host GL | proxy -> render server -> vkr -> Vulkan | QEMU Rutabaga -> backend -> Vulkan | component/FFI -> backend |
| 控制逻辑 | context/resource/SUBMIT/fence | capset/context/ring control | ASG context/ring/flush | FFI command/component callback |
| 数据传输 | VirGL command stream | shared ring/blob | ASG command stream | QEMU -> C ABI -> Rust -> component |
| 内存 ownership | GEM/backing/blob | ring + Vulkan memory/blob | ASG + GFXStream resource | Rutabaga resource table/cookie |
| 同步 | virtio fence + Host GL | ring seqno + Vulkan sync | ASG seqno + backend fence | callback + virtio fence response |
| 呈现 | Guest KMS/QEMU scanout | Guest WSI/dma-buf/scanout | native buffer/dma-buf/WSI | component 决定，QEMU 只中转 |

对应宏观入口：

- [VirGL](virgl-overview.md)
- [Venus](venus-overview.md)
- [GFXStream](gfxstream-overview.md)
- [Rutabaga](rutabaga-overview.md)

## 1. 证据基线

本文源码结论对应以下工作树提交：

| 组件 | 提交 / 版本 |
| --- | --- |
| Linux | `6b4b543fa18757297ac756a6fe6e64f506b7844a`，2026-10-25 |
| Mesa | `ec7b2f322abbcd593d600a14afe2d2773512af4e`，2026-10-26 |
| virglrenderer | `e0a1caf3b147bb26dfe524dbb85ded1177e7e49e`，2026-10-21 |
| QEMU | `d73149283205645625bfa9fe029cfb68e4c4a9ae`，2026-10-24 |
| 已知 QEMU 实验基线 | 11.1.50，见 [PHASE_0](../../PHASE_0_HOST_QEMU_VIRTIO_GPU_PATH.md) |

源码行号会随提交漂移。本文优先给函数名和层间契约，行号只用于在当前工作树
快速定位。实验结果与源码分析严格分开：本文不新增实验结论，现有运行证据仍
见各 Phase 文档和 `build/traces/`。

## 2. 先建立正确的总模型

virtio-gpu 首先是一个**虚拟 PCI 设备及其 virtqueue 协议**。VirGL、Venus、
GFXStream Vulkan、cross-domain 和 DRM native context 是这个设备上的
**capset**。一个 capset 再选择对应的 Guest 用户态驱动与 Host renderer。

因此不能说“virtio-gpu 就是 VirGL”，也不能把以下三件事混为一件事：

1. **设备语义**：Guest 看到 `1af4:1050` 的 virtio-gpu，而不是物理 NVIDIA
   或 AMD GPU 的 PCI function。
2. **渲染执行**：Vulkan/OpenGL/DRM 命令可能由 Host renderer 执行，也可能
   被转换后调用 Host 厂商驱动。
3. **呈现执行**：合成器和 scanout 可能在 Guest，也可能通过 dma-buf、
   cross-domain 或 Host compositor 完成。

```text
Guest application
  -> graphics API / guest runtime
  -> Guest UMD or vendor/KMD-specific proxy
  -> guest-visible ABI
  -> Guest DRM virtio-gpu
  -> virtqueue
  -> QEMU virtio-gpu device
  -> renderer adapter
  -> Host API or Host native driver
  -> physical GPU
```

Guest 内部的 DRM 设备始终是 virtio-gpu。Host 是否调用 NVIDIA ICD、AMD
libdrm_amdgpu、AMD libhsakmt 或软件 renderer，不改变 Guest PCI topology。

所有方案都必须回答四个独立问题：

| 平面 | 问题 |
| --- | --- |
| Control plane | 谁创建设备、context、queue、resource，并协商版本 |
| Data plane | 命令流、ring、descriptor 和执行请求如何到达 Host |
| Memory plane | Guest VA、Guest physical、Host VA、dma-buf 和 VRAM 如何对应 |
| Completion/present plane | fence 如何回来，图像如何进入 compositor 和 scanout |

请求成功、命令完成、像素正确和最终可见是四个逐级增强的验收条件。

## 3. 全路径共用的对象

本节先建立所有路径共享的设备对象模型。KMD 内部的 descriptor 分配、
response 回收、DMA sync 与 fence signal 顺序、BO 锁和 restore list 见
[共用 KMD 源码跟踪](shared-kmd-source-trace.md)。

### 3.1 Feature 与 capset

virtio feature 表示设备是否实现某类协议命令，例如 3D、resource blob、
context init、blob alignment。capset 表示某个 3D/计算命令家族的 ABI。

当前主要 capset：

| ID | Capset | Guest 客户端 | Host 后端 |
| ---: | --- | --- | --- |
| 1 | `VIRGL` | Mesa Gallium virgl | vrend / Host GL |
| 2 | `VIRGL2` | Mesa Gallium virgl | vrend / Host GL |
| 3 | `GFXSTREAM_VULKAN` | GFXStream Vulkan ICD | Gfxstream backend |
| 4 | `VENUS` | Mesa virtio Vulkan ICD | Venus proxy + `vkr` |
| 5 | `CROSS_DOMAIN` | Guest display/OS integration | cross-domain backend |
| 6 | `DRM` | Guest native-context bridge | virglrenderer native DRM |

`GET_CAPSET_INFO` 和 `GET_CAPSET` 不只返回“支持与否”，还承载协议版本、
ring 大小、格式、地址空间、硬件信息和后端能力。Guest 不能假设 Host 有某个
新字段；新字段通常只能追加在结构末尾，旧 Host 应给出零值或降级行为。

### 3.2 Context

一个 context 把 Guest 的逻辑 API 上下文绑定到 Host renderer 内的状态对象。
通常还要指定：

- `capset_id`：选择协议和 renderer。
- `num_rings`：允许的 completion/timeline 数量。
- `poll_rings_mask`：哪些 ring 需要 fence event。
- `debug_name`：trace 和 profiling 名称。

VirGL 的 context 可随文件隐式创建；Venus、GFXStream 和 native DRM 通常通过
`DRM_IOCTL_VIRTGPU_CONTEXT_INIT` 显式给出 capset 和 ring 配置。

### 3.3 Resource 与 blob

普通 2D/3D resource 是 Guest kernel 创建的 GEM object，Host renderer 通过
`RESOURCE_CREATE_2D/3D` 建对应资源，必要时通过 backing SG 暴露 Guest 页。

blob resource 支持不同的内存归属：

| `blob_mem` | 主要含义 |
| --- | --- |
| `GUEST` | 由 Guest 内存 backing，通常通过 mem entry 附加 |
| `HOST3D` | Host renderer/驱动分配 storage |
| `HOST3D_GUEST` | Host3D 分配，但有 Guest 可见/可映射带宽 |

blob flags 又描述用途，例如 mappable、shareable、cross-device。不要把
“分配发生在 Host”误解为“Guest 不能访问”，也不能把 “mappable” 误解为
“零拷贝”或“任意进程可访问”。

### 3.4 Fence

virtio-gpu 的 completion 既有 virtio control response，也有 GPU timeline
fence。控制 response 表示“设备处理完某条协议命令”，不总是表示 GPU shader
已经执行完。

典型路径：

```text
Guest submits work with fence_id
  -> Host renderer submits to API/driver
  -> Host gets native completion
  -> QEMU writes virtio fence response / event
  -> Guest virtio-gpu signals dma_fence
  -> Guest UMD returns sync/file descriptor completion
```

`ring_idx` 区分不同 hardware queue 的 timeline。把所有 workload 都当作一个
fence 序列会错误理解 Venus/GFXStream/native DRM 的并行性。

### 3.5 三种内存命名不能混用

| 名称 | 地址空间与所有者 |
| --- | --- |
| Guest VA | Guest 应用进程内的用户地址 |
| Guest physical / GPA | Guest kernel 与 virtio 设备看到的地址 |
| Host VA | QEMU 或 Host renderer 进程内地址 |
| DMA address | Host IOMMU/DMA API 给设备使用的地址 |
| GPU VA | Host 厂商驱动为 GPU 页表分配的虚拟地址 |
| VRAM | 物理 GPU 显存 |

在 API remoting 中，Guest VA 通常先经 Guest KMD 转成 mem entry，再由 QEMU
映射成 Host iovec。在 VFIO 中则完全不同：Guest 直接编程物理 PCI function，
由 IOMMU 把 Guest DMA 映射到 Host 物理内存。

## 4. VirGL/OpenGL 源码路径

### 4.1 Guest 应用和 Gallium

应用调用 `glDrawArrays`、`glDrawElements` 或 indirect draw。Mesa Gallium
frontend 把 API 状态转换为 `pipe_context` 调用，最终进入 VirGL Gallium
driver。

状态和 draw 的编码入口包括：

- [`virgl_encoder_write_cmd_dword()`](../../mesa/src/gallium/drivers/virgl/virgl_encode.c:540)
- [`virgl_encoder_draw_vbo()`](../../mesa/src/gallium/drivers/virgl/virgl_encode.c:985)

命令头是：

```text
low  16 bits: VirGL command/object opcode
high 16 bits: payload size in dwords
```

VirGL 在 Guest 用户态生成命令流，应用的一次 GL call 不必然对应一次内核
submit。命令写入 command buffer 后可以批量提交。

当 batch 需要推进时会到达：

- [`virgl_flush_eq()`](../../mesa/src/gallium/drivers/virgl/virgl_context.c:1086)
- [`virgl_drm_winsys_submit_cmd()`](../../mesa/src/gallium/winsys/virgl/drm/virgl_drm_winsys.c:954)

winsys 填 `drm_virtgpu_execbuffer`：命令指针、长度、BO handle 数组、input/
output fence、ring index。随后调用 `DRM_IOCTL_VIRTGPU_EXECBUFFER`。

### 4.2 Guest virtio-gpu KMD

内核入口是
[`virtio_gpu_execbuffer_ioctl()`](../../linux/drivers/gpu/drm/virtio/virtgpu_submit.c:478)。
读源码时按以下顺序跟踪：

1. 检查 3D feature。
2. 确保当前 `drm_file` 有 context，必要时创建。
3. 拷贝 `bo_handles` 并引用 GEM object。
4. 用 `vmemdup_user()` 把命令流复制到内核内存。
5. 解析 input fence/syncobj，等待 foreign dependency。
6. 获取 BO reservation lock，防止提交期间 backing 被破坏。
7. 调用 `virtio_gpu_submit()` 和 `virtio_gpu_cmd_submit()`。
8. 把 control request、payload、response header 组成 scatter-gather。
9. `virtqueue_add_sgs()`、kick、notify。

一个常见误区是“Guest 命令直接映射给 Host GPU”。当前 VirGL 路径中，命令
payload 至少经过一次从 Guest scatter-gather 到 QEMU 临时 buffer 的复制。

### 4.3 QEMU 设备层

QEMU 从 control virtqueue 取出 `virtio_gpu_ctrl_command`，由
[`virtio_gpu_handle_ctrl()`](../../qemu/hw/display/virtio-gpu.c:1323) 入队，
再由 [`virtio_gpu_process_cmdq()`](../../qemu/hw/display/virtio-gpu.c:1253)
分派。

VirGL 后端的 `VIRTIO_GPU_CMD_SUBMIT_3D` 处理在
[`virgl_cmd_submit_3d()`](../../qemu/hw/display/virtio-gpu-virgl.c:632)：

1. 校验 guest 给的 `size`。
2. 用 `iov_to_buf()` 复制 command payload。
3. 调用 `virgl_renderer_submit_cmd(ctx_id, buffer, size)`。

QEMU 只负责把 virtio capset ABI 送到 renderer，不需要理解每一条 GL state
或 draw 的语义。

### 4.4 virglrenderer 与 vrend

全局入口
[`virgl_renderer_submit_cmd()`](../../virglrenderer/src/virglrenderer.c:319)
按 `ctx_id` 查 context，再调用后端虚函数 `submit_cmd`。

VirGL/VirGL2 context 到达
[`vrend_decode_ctx_submit_cmd()`](../../virglrenderer/src/vrend/vrend_decode.c:2061)。
decoder 通过 opcode table 找到 handler，例如：

- [`vrend_decode_draw_vbo()`](../../virglrenderer/src/vrend/vrend_decode.c:463)
- [`vrend_draw_vbo()`](../../virglrenderer/src/vrend/vrend_renderer.c:5890)

最终调用 Host OpenGL：

```text
glDrawArrays
glDrawElements
glDrawElementsBaseVertex
glDrawArraysIndirect / glDrawElementsIndirect
```

这意味着 renderer 必须重建大量 OpenGL 状态。性能分析应分别看：

- Guest 编码和 batch 大小；
- virtqueue 通知频率；
- QEMU 复制和命令解码；
- Host GL driver 的状态校验和 draw；
- GPU 实际执行；
- resource transfer、readback 和呈现。

### 4.5 VirGL resource 与 transfer

普通 VirGL resource 可以通过 Guest backing SG 向 Host GL texture/buffer
上传。`TRANSFER_TO_HOST_3D` 和 `TRANSFER_FROM_HOST_3D` 描述 box、mip level、
stride 和 layer stride；blob 则允许 Host3D storage 或共享 mapping。

需要区分：

- **resource 创建**：建立 Host object 和 resource ID 映射。
- **context attach**：让某 context 能引用该 resource。
- **blob map**：把 Host-visible 共享窗口映射到 Guest。
- **transfer**：显式搬运数据区域。
- **flush**：通知显示路径某区域已更新。

resource 数量很多但 scanout 很少，是正常现象；不能用 scanout 数量推断
render workload。

## 5. Venus/Vulkan 源码路径

### 5.1 Guest Vulkan loader 与 ICD

Vulkan loader 按 ICD JSON 加载 Mesa virtio Vulkan driver。应用调用标准
Vulkan API，Mesa Venus client 将调用编码为 Venus protocol。

初始化关键代码：

- [`virtgpu_ioctl_context_init()`](../../mesa/src/virtio/vulkan/vn_renderer_virtgpu.c:139)
- [`vn_instance_init_ring()`](../../mesa/src/virtio/vulkan/vn_instance.c:128)
- [`vn_ring_create()`](../../mesa/src/virtio/vulkan/vn_ring.c:286)
- [`vn_ring_submit_locked()`](../../mesa/src/virtio/vulkan/vn_ring.c:631)

Venus 不把全部 Vulkan 命令都塞进 `SUBMIT_3D` payload。它创建共享 ring/blob，
把 encode 后的命令写入 ring，再通过较小的控制提交推进 tail。`SUBMIT_3D`
只有 16/24 字节并不表示 Vulkan command 只有这么小。

### 5.2 Ring 协议

`vn_ring` 的主要状态是 shared head/tail、guest write position 和 sequence。
Guest 写入 command 后：

1. 更新 ring tail。
2. 通过 `SUBMIT_3D` 通知 Host。
3. Host 消费 ring，推进 head。
4. Guest 根据 ring seqno 判断命令是否已被消费。

需要特别区分 ring consumption 和 Vulkan fence signal：

- ring head 前移表示 Host 已读取命令；
- `vkQueueSubmit` 的 fence 完成表示 Vulkan submission 完成；
- swapchain present fence 完成表示呈现引擎完成。

三者卡住时的排查处不同。

### 5.3 QEMU 与 proxy render server

QEMU 的 `SUBMIT_3D` 入口仍是 `virtio_gpu_virgl_process_cmd()`。Venus
capset 下 virglrenderer 创建 proxy context，
[`proxy_context_submit_cmd()`](../../virglrenderer/src/proxy/proxy_context.c:270)
把控制消息送给 render server。

```text
QEMU
  -> libvirglrenderer proxy client
  -> process socket / shmem
  -> virgl_render_server
  -> vkr
  -> Host Vulkan loader and ICD
```

独立 render server 进程是 attack surface 和生命周期边界，不等于安全的
硬件隔离；仍需沙箱、权限、seccomp 和资源限制。

### 5.4 Host `vkr`

render server 的 `vkr_context_init_dispatch()` 安装按 Vulkan object/type
分派的 handler，包括：

- instance/device/queue；
- device memory、buffer、image；
- descriptor、pipeline、command buffer；
- semaphore、event、fence；
- acceleration structure。

`vkr_ring.c` 消费 Guest ring，`vkr_transport.c` 负责共享传输，具体 handler
最终调用 Host Vulkan 函数。Host object handle 不是 Guest handle，两者通过
protocol object table 对应。

### 5.5 Venus memory 与 WSI

Venus 使用 `HOST3D` blob、mappable/shared blob 和可能的 dma-buf 传递：

- command/ring blob：Guest/Host 高频繁共享；
- device memory blob：承载 buffer/image；
- export/import fd：用于 WSI 或跨进程共享。

Guest 显示仍由 Guest compositor/Wayland/X11 和 virtio-gpu scanout 完成。
“Vulkan present API 成功”只表示 WSI 协议状态成功，不自动表示 Host 窗口
可见或 dma-buf modifier 正确。Gfxstream native dma-buf 的 Xid 69 记录正是
渲染路径与呈现路径不同层的实例。

## 6. GFXStream + Rutabaga 源码路径

### 6.1 Guest GFXStream ICD

GFXStream Guest Vulkan ICD 暴露标准 Vulkan entrypoints，但实现不是 Guest
本地 Vulkan driver。它把调用序列化到 GFXStream 协议。

关键路径：

- [`createVirtioGpuAddressSpaceStream()`](../../mesa/src/gfxstream/guest/GoldfishAddressSpace/VirtioGpuAddressSpaceStream.cpp:68)
- [`DrmVirtGpuDevice::init()`](../../mesa/src/gfxstream/guest/platform/drm/DrmVirtGpuDevice.cpp:147)
- [`DrmVirtGpuDevice::createBlob()`](../../mesa/src/gfxstream/guest/platform/drm/DrmVirtGpuDevice.cpp:309)

初始化步骤：

1. 查 `RESOURCE_BLOB`、`HOST_VISIBLE`/guest-handle 等 param。
2. 从 GFXStream capset 读取 ring size、buffer size、alignment。
3. 创建 `HOST3D` mappable blob。
4. 在 blob 上发送 `GFXSTREAM_CONTEXT_CREATE`。
5. mmap blob，把前半部分解释为 `asg_ring_storage`，后半部分作为 command
   buffer。
6. 后续 Vulkan 命令写入 address-space stream。

GFXStream ring 是另一套协议，不要拿 Venus ring 结构直接套用。

### 6.2 Vulkan encoder 与资源跟踪

Guest 入口在 `mesa/src/gfxstream/guest/vulkan/` 和 `vulkan_enc/`：

- `gfxstream_vk_cmd.cpp` 处理 command buffer；
- `VkEncoder` 序列化 Vulkan call；
- `ResourceTracker` 跟踪 handle、memory、image、format 和 WSI；
- `vkQueueFlushCommandsGOOGLE_encode_impl.cpp.inl` 反映 queue flush 的
  seqno 和 command stream 提交。

GFXStream 有大量 descriptor/image emulation。例如格式能否导出 VirtGpu
resource、modifier 能否作为 color attachment，都是单独的 feature 决定，
不能仅从一个 enum 映射表推断整条路径可用。

### 6.3 QEMU Rutabaga 适配层

QEMU 设备是 `virtio-gpu-rutabaga` / `virtio-vga-rutabaga`。
[`virtio_gpu_rutabaga_process_cmd()`](../../qemu/hw/display/virtio-gpu-rutabaga.c:777)
把 virtio 命令转换为 Rutabaga FFI 调用：

| virtio 命令 | Rutabaga FFI |
| --- | --- |
| `CTX_CREATE/DESTROY` | `rutabaga_context_*` |
| `SUBMIT_3D` | `rutabaga_submit_command` |
| `RESOURCE_CREATE_3D` | `rutabaga_resource_create_3d` |
| `RESOURCE_CREATE_BLOB` | `rutabaga_resource_create_blob` |
| `RESOURCE_MAP/UNMAP_BLOB` | `rutabaga_resource_map/unmap` |
| attach/detach backing | `rutabaga_resource_*_backing` |

`rutabaga_gfx_ffi` 是 C ABI，`rutabaga_gfx` 是安全 Rust 包装，
GFXStream backend 最终进入 `libgfxstream_backend`。固定实验版本用
`rutabaga_gfx_ffi 0.1.3` 和 `rutabaga_gfx = 0.1.3`，说明 Rust crate
版本也不是随便兼容；Cargo 次版本解析差异曾导致 ABI/构建失败。

### 6.4 Host GFXStream backend

固定 GFXStream 提交中，Host Vulkan 的主要源码位置是：

- `host/virtio_gpu_gfxstream_renderer.cpp`
- `host/renderer_impl.cpp`
- `host/renderer_operations.cpp`
- `host/vulkan/vk_decoder.cpp`
- `host/vulkan/vk_sub_decoder.cpp`
- `host/vulkan/external_memory.cpp`
- `host/vulkan/display_vk.cpp`

Host decoder 把 Guest command stream 还原为 Host Vulkan 调用，维护 Host
handle 映射、external memory 和 compositor。GFXStream 的名称、版本、feature
查询和错误状态可能来自 Host Vulkan，因此 Guest 看到 NVIDIA 名称仍不代表
PCI passthrough。

### 6.5 GFXStream WSI 是独立验收面

四条常见呈现路径：

1. Guest X11/Wayland 使用 dma-buf，Guest compositor 合成后交 Guest KMS。
2. Guest Wayland 使用 `wl_shm`，Host 渲染后回读/拷贝到 Guest shared memory。
3. Host-side cross-domain，把 Guest Wayland/X11 请求和 fd 转给 Host
   compositor。
4. Host scanout/blit 到 QEMU display。

每一层都有格式、modifier、stride、plane、acquire/release fence 和 ownership
约定。swapchain image 创建成功不等于 color attachment 合法，也不等于
present 后内容可见。

## 7. 扩展对照：Native DRM context 源码路径

独立宏观和逐函数文档见
[native-drm-overview.md](native-drm-overview.md) 与
[native-drm-source-trace.md](native-drm-source-trace.md)。

### 7.1 “Native”在这里不是 PCI passthrough

Native DRM context 转发的不是 OpenGL/Vulkan 高层 API，而是某个 Host 厂商
DRM 驱动附近的 UAPI/命令 ABI。它仍由 Guest virtio-gpu 驱动承载，仍通过
virtqueue 和 Host renderer。

```text
Guest Mesa/vendor UMD
  -> guest native-context shim
  -> virtio-gpu DRM capset (6)
  -> QEMU
  -> virglrenderer native context
  -> Host libdrm / vendor KMD
  -> physical GPU
```

Guest 不拥有物理 BAR、PCI config、MSI-X 向量或 IOMMU group。Native context
的驱动必须遵守 Guest/Host ABI 兼容、资源映射和 fence 契约。

### 7.2 Guest 侧 vDRM

[`vdrm_virtgpu_connect()`](../../mesa/src/virtio/vdrm/vdrm_virtgpu.c:371) 是 Guest
native context 的通用桥：

1. `GET_CAPS` 查询 capset 6，检查 `context_type` 是否匹配目标驱动。
2. `CONTEXT_INIT` 设置 `VIRTGPU_DRM_CAPSET_DRM` 和 ring 数。
3. 创建 `HOST3D + MAPPABLE + blob_id=0` 的 16 KiB 控制 shmem。
4. `VIRTGPU_MAP` + `mmap` 让 Guest 用户态看到请求/响应 buffer。
5. 提供 `bo_create`、`bo_map`、dma-buf import/export、wait 和 execbuf
   回调。

请求不是原生厂商 ioctl 原样穿越。`struct vdrm_ccmd_req` 带 `cmd/len/seqno/
rsp_off`，响应头带 `len`。Guest 用户态把多条请求批量写入 command buffer，
再由 `VIRTGPU_EXECBUFFER` 提交。

### 7.3 AMDGPU Guest shim

AMD native context 在 Mesa 中表现为一个可替换 libdrm_amdgpu 的 shim：

- [`amdvgpu_device_initialize()`](../../mesa/src/amd/common/virtio/amdgpu_virtio_device.c:97)
- [`amdgpu_virtio_proto.h`](../../mesa/src/amd/common/virtio/amdgpu_virtio_proto.h:12)

Guest shim 把上层原本使用 libdrm 的代码转成 AMDGPU control commands：

- query info；
- GEM new；
- BO VA mapping；
- CS submit；
- metadata query/set；
- context create/destroy；
- VMID reserve；
- fence status query。

它还要维护 Guest GPU address space、host object mapping 和 virtio ring
seat。`AMDGPU_CCMD_CS_SUBMIT` 是实际提交路径，ring 0 留给 CPU/host sync。

### 7.4 QEMU 和 virglrenderer

QEMU 的 `virtio-gpu-gl` 只有开启 `drm_native_context=on`、Host EGL 可用且
virglrenderer 足够新时才使能 `VIRGL_RENDERER_DRM`。见
[`virtio-gpu-gl.c`](../../qemu/hw/display/virtio-gpu-gl.c:166) 和
[`virtio_gpu_virgl_init()`](../../qemu/hw/display/virtio-gpu-virgl.c:1471)。

virglrenderer 在 `drm_renderer_init()` 中探测 Host render node 并选择：

| context type | Backend |
| ---: | --- |
| 1 | MSM |
| 2 | AMDGPU |
| 3 | i915 |
| 4 | Asahi |
| 5 | Panfrost |

通用 [`drm_context_submit_cmd()`](../../virglrenderer/src/drm/drm_context.c:178)
负责：

- 校验 command header、长度和 alignment；
- 按 opcode 查 handler；
- 用 shadow buffer 防止 handler 直接信任 Guest 输入；
- 校验 response offset，防止 Host 越界写 Guest shmem；
- 发布 response seqno。

AMDGPU backend 在 `amdgpu_renderer_probe()` 中查询 Host GPU 信息、VA 上限、
alignment 和硬件信息；在 `amdgpu_renderer_create()` 中为每个 Guest
application/context 建立独立 Host `amdgpu_device`、timeline 和 ring 映射。

### 7.5 Resource 与 fence

AMDGPU native backend 支持从 Guest resource 导出 dma-buf 后，在 Host 用
`amdgpu_bo_import()` 导入；Host object 分配可通过 blob ID 导出为 dma-buf。
这种设计把资源身份和存储放在 Host amdgpu 对象上，Guest 仍只看到 virtio
resource handle。

`amdgpu_renderer_submit_fence()` 把每个 Guest ring 映射到 Host timeline。
ring 0 是 Guest 与 Host CPU 的同步 ring，可直接完成；其他 ring 由 Host
amdgpu fence FD 驱动 completion。

### 7.6 为什么 native context 仍然强耦合

它绕过高层 API 解码，但没有消除虚拟化：

- Guest UMD 必须生成本协议期望的命令；
- Guest 和 Host 驱动版本/feature 必须兼容；
- Guest VA 与 Host GPU VA 要协商；
- resource backing、dma-buf、cache 属性和 flush 必须一致；
- Host renderer 是信任边界；
- 错误、reset 和迁移仍由虚拟设备管理。

其性能上限可能高于 API remoting，但并非天然零开销，也不具备 VFIO 的硬件
隔离和完整 PCI fault semantics。

## 8. 四条路径横向对照

| 维度 | VirGL | Venus | GFXStream | Rutabaga |
| --- | --- | --- | --- | --- |
| Guest 高层 API | OpenGL | Vulkan | Vulkan/GLES/composer 可组合 | 由所选 component 决定 |
| Guest UMD | Mesa VirGL | Mesa Venus | GFXStream ICD | 无 Rutabaga UMD |
| Guest 协议主体 | VirGL command | Venus protocol + ring | GFXStream command stream | 透传所选 capset 的 command bytes |
| QEMU adapter | virglrenderer API | virglrenderer API | QEMU Rutabaga | QEMU Rutabaga 本身 |
| Host 执行 | vrend -> GL | vkr -> Vulkan | GFXStream backend -> Vulkan | component -> backend/API |
| 共享内存 | backing/blob | ring/blob/dma-buf | ASG blob/dma-buf | FFI 传递 iov/blob/handle |
| 同步 | virtio fence + GL sync | ring seqno + Vulkan sync | ASG seqno + backend fence | FFI callback + virtio fence |
| 驱动耦合 | 中 | 中 | 中，且有协议版本 | 框架稳定，component 强耦合 |
| Guest API 完整性 | Host GL 能力约束 | Host Vulkan 能力约束 | 后端 feature/emulation 约束 | 完全取决于 component |
| 呈现复杂度 | Guest WSI + scanout | Guest WSI + scanout | dma-buf/shared-memory/cross-domain | component 决定，QEMU 只中转 |

Rutabaga 与其他三项不是完全同级的 renderer 对比：它自己不解码 Vulkan 或
OpenGL。它提供 QEMU-to-component 的统一控制与资源模型，当前选择 GFXStream
时，实际图形执行仍发生在 `libgfxstream_backend`。这张表把 Rutabaga 单列，
是为了明确用户要求中的第四个架构边界，而不是暗示它有独立 Guest API。

Native DRM context 仍是 virtio-gpu 的重要扩展，但不在本轮 `N = 4` 的四条
主路径中。它的源码链保留在第 7 节作为架构对照。

## 9. 内存所有权矩阵

| 场景 | Guest 谁拥有 | Host 谁拥有 | 释放触发的关键点 |
| --- | --- | --- | --- |
| virgl command buffer | 用户态临时 buffer -> KMD copy | QEMU temp buffer -> renderer decode | submit response + fence |
| Venus ring blob | Guest mapping + resource | renderer/proxy shared object | resource unref/context destroy |
| Venus device blob | Guest GEM/blob handle | Host Vulkan memory | Guest free 与 Host unref |
| GFXStream ASG blob | Guest mapped ring storage | backend mapped resource | context/resource teardown |
| Rutabaga resource/context | Guest resource/context ID | Rust resource table + component object | FFI destroy + backend completion |
| Native shmem | Guest `dma-buf`/GEM handle | vdrm shmem mapping | context/vdrm close |
| Native object | Guest resource ID/handle | Host BO/dma-buf | detach/unref and fence |
| userptr proposal | Guest user pages, long-term pinned | QEMU iovec/sg -> renderer | Guest resource unref + Host completion |

审查任何新 feature 时，至少画两条时序：

1. 正常 create -> use -> unref；
2. 中途失败、Host crash、Guest teardown、reset 或 hibernation。

资源泄漏通常来自只设计第一条路径。

## 10. 如何按症状定位

### 10.1 Guest 没有新 capset

检查顺序：

1. Host renderer 是否编译了后端；
2. QEMU device property 是否开启；
3. virglrenderer 初始化 flag 是否设置；
4. `GET_CAPSET` 是否返回有效 size/version；
5. QEMU 是否把 capset id 放入 config；
6. Guest KMD 是否读到对应 capset；
7. Guest ICD 是否真的被 loader 选中。

不要从 `lspci` 的 NVIDIA 名称推断 Gfxstream capset 存在。

### 10.2 Device 创建成功，submit 卡住

依次区分：

- 控制命令是否被 QEMU 消费；
- `SUBMIT_3D` 是否到达 renderer；
- ring tail 是否被 Host 看到；
- Host API 是否提交；
- native fence 是否完成；
- virtio fence response 是否写回；
- Guest dma_fence/syncobj 是否 signal；
- Guest UMD 是否在等待错误对象。

Venus 的 24 字节 submit 成功只证明控制包被处理，不证明 Vulkan fence 已
完成。

### 10.3 离屏像素正确，但窗口错误

查 WSI：

1. 应用选择什么 surface format；
2. swapchain image 的 tiling/modifier/usage；
3. export/import 的 fd 类型；
4. Guest compositor 是否支持相应 dma-buf modifier；
5. acquire/release 和 present fence；
6. Guest KMS/QEMU scanout/Host compositor 是否完成。

现有 GFXStream native WSI 诊断展示了“submit 正确、swapchain 创建成功、
但 native image render target 非法”的精确分层。

### 10.4 Fence 偶发超时

优先检查 timeline/ring 映射和 retry 逻辑，而不是直接怀疑 GPU：

- 是否使用正确 `ring_idx`；
- ring seqno 与 Vulkan/DRM native fence 是否混用；
- QEMU 是否在 unmap 时暂停队列；
- fence response 是否因为 error 丢失；
- Guest 和 Host 的 protocol 版本是否匹配；
- Host API validation 是否报告同步错误。

## 11. 建议阅读顺序

第一次阅读：

1. 本文第 2、3 节；
2. `PHASE_1_GUEST_DRM_RESOURCES.md` 的 resource/blob 结构；
3. `PHASE_2_QEMU_DEVICE_BOUNDARY.md` 的 virtqueue、mapping 和 fence；
4. 本文 VirGL 路径；
5. `PHASE_0_HOST_QEMU_VIRTIO_GPU_PATH.md` 的 trace 对照。

第二次阅读：

1. 本文 Venus 路径；
2. `PHASE_3_VENUS_VULKAN_PATH.md`；
3. `KVM_VENUS_DIAGNOSIS.md`；
4. 单独阅读 `vn_ring.c`、`proxy_context.c` 和 `vkr_ring.c`。

第三次阅读：

1. 本文 GFXStream 路径；
2. `PHASE_4_GFXSTREAM_RUTABAGA.md`；
3. `GFXSTREAM_NATIVE_WSI_XID69_DIAGNOSIS.md`；
4. GFXStream Guest `ResourceTracker` 与 Host `vk_decoder`。

第四次阅读：

1. 本文 native DRM 路径；
2. `mesa/src/virtio/vdrm/`；
3. `mesa/src/amd/common/virtio/`；
4. `virglrenderer/src/drm/drm_context.c`；
5. `virglrenderer/src/drm/amdgpu/amdgpu_renderer.c`。

最后阅读社区专题
[Userptr/ROCm compute](userptr-rocm-compute.md)
和跨方案专题
[Platform comparison](platform-comparison.md)。

## 12. 从“看懂”到“专家”的检查题

源代码级问题：

1. 为什么 VirGL command buffer 中的 draw 数量不等于
   `DRM_IOCTL_VIRTGPU_EXECBUFFER` 次数？
2. Venus 为什么能用很小的 `SUBMIT_3D` 承载大量 Vulkan 命令？
3. Venus ring head 前移为什么不能直接等价于 Vulkan fence signal？
4. GFXStream ring blob 的前后两个区域分别是什么？
5. Rutabaga 为什么需要 C FFI，而不是让 QEMU 直接链接 Gfxstream C++？
6. Native DRM context 为什么仍需要 Guest virtio-gpu 驱动？
7. `VIRTGPU_BLOB_MEM_GUEST`、`HOST3D`、`HOST3D_GUEST` 的 ownership 有何
   不同？
8. 普通 resource backing、blob map 和 dma-buf export 各自解决什么问题？
9. 为什么 command response 和 GPU fence 不能统一成一个概念？
10. reset 时为什么必须处理尚未完成的 unmap/unref/fence？

故障分析问题：

11. `vulkaninfo` 能看到设备但 `vkcube` 超时，下一步应查哪三个平面？
12. 离屏 clear/readback 通过但 native swapchain 黑屏，哪些条件仍未证明？
13. Guest 窗口显示 Host GPU 名称，怎样用 PCI ID、capset 和 renderer
   trace 否定 passthrough？
14. Native context 能打开 Host render node，是否意味着 Guest 能直接访问
  物理 BAR？
15. 如何证明一个 blob 是 Host3D storage，而不是 Guest shmem 的拷贝？

架构判断题：

16. 哪种路径适合跨厂商 Linux Guest，哪种适合同厂商 Host/Guest 紧耦合？
17. API remoting 的 live migration 需要保存哪些 Guest visible state？
18. 为何“更低层转发”不自动等于“更高安全隔离”？
19. 哪些 workload 主要受 GPU 限制，因此虚拟化协议成本不显著？
20. 设计新的 capset 时，如何处理版本、未知字段、failure cleanup 和
    malicious Guest input？

能不用查文档回答这些问题，并能指出对应源码和 trace，才达到源码专家级。
