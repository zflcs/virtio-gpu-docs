# Virtio-GPU 路径三：GFXStream

函数级源码链见 [gfxstream-source-trace.md](gfxstream-source-trace.md)。

## 1. 一句话定位

GFXStream 是面向多图形 API 的 **graphics API streaming** 系统。Guest
GFXStream ICD/encoder 把 Vulkan 或 Android 图形调用序列化到 ASG ring；在
QEMU 中通常由 Rutabaga 承载，再进入 `libgfxstream_backend`，最终调用 Host
Vulkan 或其他 Host 图形后端。

```text
Guest Vulkan/GLES application
  -> Guest GFXStream driver/ICD
  -> GFXStream command stream
  -> ASG ring in virtio-gpu blob
  -> Guest DRM virtio-gpu
  -> virtqueue
  -> QEMU virtio-gpu-rutabaga
  -> Rutabaga component
  -> libgfxstream_backend
  -> Host Vulkan
  -> physical GPU
```

GFXStream 和 Rutabaga 不是同一层：GFXStream 是 API streaming 后端，Rutabaga
是 Host VMM 侧的组件适配和 FFI 层。本文讲 GFXStream 本身；Rutabaga 的职责
在单独文档中展开。

## 2. 系统层级

| 层 | GFXStream 中的角色 |
| --- | --- |
| Guest 应用 | Vulkan、GLES、Android graphics/composer workload |
| Guest UMD | GFXStream encoder/ICD |
| Guest-visible ABI | virtio-gpu capset + GFXStream command stream |
| Guest KMD | Linux virtio-gpu |
| Data transport | virtio-gpu resource/blob + ASG ring |
| VMM adapter | QEMU Rutabaga（部署相关） |
| Host renderer | GFXStream backend |
| Host API | Vulkan、GLES 或其他 component |
| Hardware | Host 物理 GPU |

## 3. Guest UMD 如何工作

GFXStream Guest UMD 不是直接向 Host Vulkan API 透传裸 handle。它：

1. 暴露标准 Vulkan/GLES entrypoints 给 Guest 应用。
2. 在 Guest 中分配并登记包装对象和 resource ID。
3. 把 API 参数序列化为 GFXStream 协议。
4. 对部分 object/format/extension 做 emulation 或 capability masking。
5. 将 Vulkan command buffer stream 写入 GFXStream command buffer。
6. `QueueSubmit/QueueFlush` 时把 command bytes 放进 ASG ring。
7. 等待 ring seqno 和 Host fence completion。

GFXStream 是面向 Android 生态演化出来的多 API 框架，设计目标不只是一组
desktop Vulkan 调用。descriptor virtualization、Android native buffer、
gralloc 和 composer 等概念会影响 Guest UMD。

## 4. Guest KMD 如何工作

Guest KMD 与 Venus 有相似的 virtio-gpu 基础职责，但 GFXStream 使用自己的
capset 和 ring 参数：

- `RESOURCE_BLOB` 创建 ASG/command blob；
- `CONTEXT_INIT` 设置 GFXStream capset 和 ring 数量；
- `SUBMIT_3D` 提交 command stream 或控制消息；
- resource attach/detach 管理对象可见性；
- fence response 推进 Guest completion；
- host-visible mapping 允许 Guest 写 ring。

KMD 不理解 Vulkan object，也不解释 GFXStream opcode。它只确保 command
bytes、resource handles 和 backing 被安全传输。

## 5. 如何使用 Host GPU

Host GFXStream backend 解码 Guest command stream，并把协议操作转成 Host
Vulkan 调用。Host backend 管理：

- Host VkInstance/Device/Queue；
- image、buffer、memory、sampler、pipeline；
- external memory 和 dma-buf；
- descriptor emulation；
- compositor/display；
- Host fence 与 device-lost 处理。

GFXStream 可以把 Host Vulkan 的 device name、vendor/device ID、driver 名称
等属性转发给 Guest。因此 Guest 里显示 NVIDIA/AMD 名称不代表物理 PCI 直通。

## 6. 这是什么形式的虚拟化

GFXStream 是：

- **API streaming / API remoting**；
- Guest 与 Host 拆分图形栈的 split driver；
- 面向多 API/Android 生态的设备虚拟化；
- 可共享 Host GPU renderer 的软件设备模型。

它不是 VFIO whole-GPU passthrough，也不是 SR-IOV VF。Host renderer 和协议
解析器是主要信任边界。

## 7. 控制逻辑

### 7.1 Capability 协商

Guest 查询：

- virtio-gpu 3D/blob/context features；
- GFXStream capset 版本和 size；
- ring/buffer size、blob alignment；
- Vulkan API/device/extension feature；
- Host-supported formats 和 external-memory 能力。

这些信息决定 Guest 是否启用 Vulkan ICD、是否可用 dma-buf WSI，以及是否走
emulation 路径。

### 7.2 Context 与 ring

Guest 创建 Host3D mappable blob。blob 前半部分承载 ASG ring storage，后半
部分作为 command buffer；`GFXSTREAM_CONTEXT_CREATE` 把 resource ID 与该
context 绑定。ring 配置包含 guest write position、host consumed position
和 transfer mode。

### 7.3 Resource 与 memory

GFXStream 资源可能是：

- ASG ring/control blob；
- Host3D Vulkan memory；
- Guest-visible shared memory；
- dma-buf/external memory；
- Android native buffer/gralloc。

Guest 资源 ID 与 Host backend resource table 是两套命名。格式、modifier、
usage 和 memory properties 需要 backend 逐一转换。

### 7.4 Completion

GFXStream command stream 有 sequence number。Guest 推进 ring 后，Host 消费
command 并产生 fence/completion；Guest `QueueSubmit` 的 Vulkan fence 只在
Host submission 完成后 signal。present 还可能有独立的 acquire/release/
present fence。

## 8. 数据如何传输

主数据面是 ASG ring：

```text
Guest API encoder
  -> GFXStream command bytes
  -> Guest address-space stream
  -> mapped ASG ring/blob
  -> virtio-gpu notification
  -> QEMU/Rutabaga
  -> GFXStream decoder
  -> Host Vulkan
```

少量控制信息通过 `SUBMIT_3D` 或 control command 传输；大量 Vulkan command
bytes 位于共享 blob/ring 内。resource create/transfer、blob map/unmap 和
attach/detach 负责让 Host 能解析 command 引用的对象。

呈现数据面可能分叉：

- Host 渲染后回读/复制到 Guest `wl_shm`；
- Host 导出 dma-buf 给 Guest compositor；
- Guest compositor 合成后交 Guest KMS/QEMU scanout；
- 通过 cross-domain 把 Guest display protocol 交给 Host compositor。

## 9. 与 Venus 的核心区别

| 维度 | Venus | GFXStream |
| --- | --- | --- |
| 来源生态 | Mesa virtio Vulkan | Android/Cuttlefish/GFXStream |
| Guest 协议 | Venus protocol + ring | GFXStream ASG command stream |
| Host renderer | `vkr` in virglrenderer | `libgfxstream_backend` |
| QEMU adapter | virglrenderer proxy | Rutabaga |
| API 范围 | 以 Vulkan 为主 | Vulkan/GLES/composer 等多 API 方向 |
| 内存/display | Venus blob/dma-buf/WSI | GFXStream resource/gralloc/dma-buf/WSI |

不能把“都是 Vulkan”当作两条路径等价：它们的 object tracking、descriptor
emulation、WSI 和版本契约不同。

## 10. 性能模型

主要成本：

- Guest command serialization；
- GFXStream ring 写和同步；
- QEMU/Rutabaga FFI 和 command copy；
- Host decoder 的 object lookup；
- descriptor/image/format emulation；
- Host Vulkan driver validation/compile；
- external memory import/export 与 compositor copy；
- WSI 的 readback、blit 或 dma-buf barrier。

GFXStream 的 ring 提交不保证 zero-copy；`wl_shm` 兼容 WSI 明确包含 CPU
回读/拷贝，native dma-buf 也有格式、modifier 和同步成本。

## 11. 安全与隔离

审查重点：

- Guest 能否伪造 object ID、resource ID、command length；
- Host decoder 是否严格校验 pointer/offset/array count；
- descriptor emulation 是否泄漏其他 context 的 handle；
- image/format modifier 是否可触发非法 Host GPU 操作；
- Android/gralloc/composer 共享缓冲区生命周期；
- backend 进程与 QEMU 的权限和 attack surface；
- device lost 是否被正确转成 Guest error，而非 Host-use-after-free。

GFXStream 的多 API 能力意味着协议和 emulation 面较大，安全审查必须覆盖
实际启用的 component，而不是只看 Vulkan 部分。

## 12. 优点、缺点与适用场景

优点：

- Android/Cuttlefish 生态成熟；
- 多 API 和 compositor 方向扩展性强；
- 能复用多种 Host graphics backend；
- 支持 native buffer、dma-buf 和 WSI 组合。

缺点：

- 协议/API 面大；
- Guest/Host backend 版本强耦合；
- native WSI 的格式、modifier、fence 调试困难；
- Rutabaga/QEMU 当前有 migration blocker；
- 不同 component 的支持矩阵不一致。

适合 Android 虚拟化、GFXStream/Cuttlefish、多 API streaming 与图形兼容性
研究；不适合把“能创建 Vulkan device”当作完整显示链已通过。

## 13. 关键源码入口

Guest：

- `mesa/src/gfxstream/guest/GoldfishAddressSpace/VirtioGpuAddressSpaceStream.cpp`
- `mesa/src/gfxstream/guest/platform/drm/DrmVirtGpuDevice.cpp`
- `mesa/src/gfxstream/guest/vulkan/`
- `mesa/src/gfxstream/guest/vulkan_enc/`

Host adapter：

- `qemu/hw/display/virtio-gpu-rutabaga.c`
- `rutabaga_gfx-0.1.3/src/gfxstream.rs`

Host backend（固定 GFXStream commit）：

- `host/virtio_gpu_gfxstream_renderer.cpp`
- `host/renderer_impl.cpp`
- `host/vulkan/vk_decoder.cpp`
- `host/vulkan/vk_sub_decoder.cpp`
- `host/vulkan/external_memory.cpp`
- `host/vulkan/display_vk.cpp`

## 14. 宏观检查题

1. GFXStream 与 Rutabaga 各做什么？
2. Guest GFXStream ICD 为什么仍需要 virtio-gpu KMD？
3. ASG ring 的前后两个区域分别承载什么？
4. GFXStream 如何把 Guest Vulkan object 映射到 Host Vulkan object？
5. GFXStream 与 Venus 的 ring 协议是否可互换？
6. 为什么 Guest 显示 Host GPU 名称不代表 VFIO？
7. GFXStream resource create/transfer 与 Vulkan memory allocation 有何关系？
8. swapchain 创建成功、离屏 clear 成功和可见 present 分别证明了什么？
9. `wl_shm` 与 dma-buf WSI 的数据路径差异是什么？
10. 为什么 GFXStream 需要单独审查 Android/gralloc/composer 路径？
