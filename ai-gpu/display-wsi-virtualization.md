# GPU 显示、Compositor、WSI 与 VDI 虚拟化

## 1. 为什么显示是独立平面

GPU work 正确只证明：

```text
shader / draw / dispatch completed
```

用户看到画面还需要：

```text
render target
  -> compositor sampling
  -> scanout / flip
  -> Host display backend or remote encoder
  -> physical connector / client
```

任一层失败都可能表现为黑屏、旧帧、撕裂或只显示光标，即使离屏渲染像素完全正确。

## 2. DRM/KMS 对象

| 对象 | 作用 |
| --- | --- |
| Framebuffer | 引用 BO、format、modifier、plane/stride/offset |
| Plane | primary/cursor/overlay 图像层 |
| CRTC | 扫描时序、active mode、flip |
| Connector | 输出、EDID、hotplug |
| Encoder | 连接 CRTC 和 connector（真实硬件路径） |

虚拟 GPU 可以模拟其中部分对象，但用户可见语义仍需一致。

## 3. Virtio-GPU 显示 UAPI

### 3.1 Display Info

`VIRTIO_GPU_CMD_GET_DISPLAY_INFO` 返回 scanout 数量、启用状态、矩形和 EDID。

Guest KMS 用它构造 connector/mode；Host display resize 可通过
`VIRTIO_GPU_EVENT_DISPLAY` 通知 Guest。

### 3.2 EDID

`VIRTIO_GPU_F_EDID` 和 `GET_EDID` 提供 output 的：

- physical size；
- preferred resolution；
- refresh；
- output name。

QEMU
[`virtio_gpu_base_generate_edid()`](../../qemu/hw/display/virtio-gpu-base.c:57)
根据 QEMU output 配置生成 EDID。

EDID 改变会导致 Guest modeset。

### 3.3 Set Scanout

普通 scanout：

```text
VIRTIO_GPU_CMD_SET_SCANOUT
  resource_id, scanout_id, rect
```

Blob scanout：

```text
VIRTIO_GPU_CMD_SET_SCANOUT_BLOB
  resource_id, format, width/height, strides, offsets
```

它们把 Guest resource 绑定到 Host scanout。

### 3.4 Resource Flush

`RESOURCE_FLUSH` 表示 Guest 已更新 scanout resource 的某矩形。

它和以下 fence 不同：

- GPU render fence：work 已完成；
- transfer fence：数据已上传/回读；
- scanout bind：resource 已成为 scanout；
- resource flush：Host display backend 应更新该区域；
- present fence：compositor/display 已允许图像复用。

### 3.5 Cursor

cursor 通过独立 virtqueue：

- update 图像/hotspot；
- move 坐标；
- 清除 cursor。

cursor 小、更新频繁，通常有独立路径。cursor 更新成功不代表 primary plane 正确。

## 4. Guest DRM Plane/CRTC

[`virtgpu_display.c`](../../linux/drivers/gpu/drm/virtio/virtgpu_display.c:64)
实现：

- framebuffer funcs；
- CRTC atomic enable/disable/flush；
- connector/EDID/modes；
- primary + cursor plane。

[`virtgpu_plane.c`](../../linux/drivers/gpu/drm/virtio/virtgpu_plane.c:243)
的 primary update：

1. 判断 CRTC 是否 active；
2. 合并 damage；
3. dumb buffer 先上传；
4. resource/fb/源矩形变化时 `SET_SCANOUT`；
5. `RESOURCE_FLUSH`；
6. 可能等待 plane fence。

### 4.1 Modeset 与 plane update

源码注释指出 virtio-gpu 不能在 plane update 中同时完成 modeset 和 plane update；
CRTC atomic flush 会把这些操作排序。

### 4.2 Damage

damage clips 可减少 Host 刷新，但如果 fb 已切换且是 dumb buffer，必须忽略
damage 并做完整上传，因为旧内容没有被保证复制。

## 5. Host 显示后端

QEMU `qemu_console_*` 抽象 Host backend：

- 2D pixman surface；
- Host GL/EGL scanout texture；
- dma-buf；
- text console；
- resize/invalidate。

Host backend 可以是本地 GTK/SDL、headless、远程协议或 compositor。

Guest 画面正确不代表某 Host backend 的 resize、GL context 或 dmabuf 导入正确。

## 6. Virtio scanout resource 类型

### 6.1 2D resource

Host 使用 pixman image/surface。路径简单，适合 dumb/fbcon/2D。

### 6.2 VirGL texture

`SET_SCANOUT` 可把 renderer texture 绑定 `qemu_console_gl_scanout_texture`。

Host GL completion、flush 和 texture lifetime 必须正确。

### 6.3 Blob dma-buf

`SET_SCANOUT_BLOB` 将 blob 转为 framebuffer 信息，Host 可 import dmabuf 给 display
backend。

需要：

- format；
- stride/offset；
- plane 布局；
- dma-buf ownership；
- GL/display 支持。

当前 virtio DRM mode config 设置 `fb_modifiers_not_supported=true`；这意味着
其内核显示接口不能假设任意 DRM modifier 可传。

## 7. WSI 基础

WSI（Window System Integration）把 GPU image 交给 compositor：

```text
application rendering
  -> acquire image
  -> render/copy
  -> release/present
  -> compositor sampling
  -> scanout/display
```

WSI 的核心不是“创建 swapchain”，而是：

- image ownership 转移；
- acquire/release fence；
- format/modifier；
- compositor support；
- scanout/remote protocol；
- image 回收 timing。

## 8. WSI 路径

### 8.1 Wayland

常见：

- dma-buf + format/modifier；
- Linux explicit sync；
- surface commit；
- acquire/release；
- compositor 合成；
- DRM/KMS scanout。

虚拟化可能把 WSI 放在：

- Guest compositor -> Guest virtio scanout；
- Host renderer 导出 dma-buf -> Guest compositor；
- cross-domain -> Host compositor；
- `wl_shm` CPU shared memory readback。

### 8.2 X11 DRI3

需要：

- X11 pixmap/fd import；
- dma-buf；
- Present extension；
- compositor/window manager；
- X server DRM auth。

离屏 Vulkan 成功不证明 DRI3 present 可用。

### 8.3 Android

常见 AHardwareBuffer/gralloc/dma-buf：

- vendor format/modifier；
- gralloc usage；
- SurfaceFlinger compositor；
- Host/tunneled presentation。

GFXStream 常处理这些 Android-specific external memory 契约。

## 9. Venus WSI

Venus Guest 是标准 Vulkan application/ICD，Host 是 `vkr`。

路径可能：

- Host allocation；
- Guest blob；
- dma-buf import/export；
- udmabuf/GBM linear buffer；
- shared memory host copy。

必须匹配：

- `VkImage` format/usage；
- tiling/modifier；
- memory requirements；
- acquire/release semaphore/fence；
- compositor format support。

### 9.1 False zero-copy

以下路径都能“显示”：

- Host 渲染后回读到 Guest；
- Host export dma-buf；
- Guest render 到共享 blob；
- CPU `wl_shm` copy。

只有确认 producer/consumer 都直接访问同一 backing 且同步/格式正确，才能讨论
zero-copy。

## 10. GFXStream/Rutabaga WSI

GFXStream 的 ResourceTracker 同时处理：

- Guest Vulkan image；
- external memory；
- color attachment usage；
- DRM modifier；
- Host resource table；
- native buffer/host copy。

WSI 常见两类：

1. `wl_shm`：Host 回读/复制到 Guest shared memory；
2. native dma-buf：Host 导出/import 图像给 Guest compositor。

`wl_shm` 能显示正确但明确含 CPU copy；native dma-buf 性能更好，但受格式、
modifier、usage 和 compositor 支持影响。

现有仓库 native WSI 诊断只作为历史运行证据，不在本轮重新验证。

## 11. Native DRM context/Display

Native renderer 的资源可：

- 作为 virtio blob；
- export/import dma-buf；
- 走 `SET_SCANOUT_BLOB`；
- 走 Host GL scanout texture。

Native 3D submission 正确仍需独立验证 scanout：

- native BO format；
- stride/offset；
- dmabuf import；
- display backend；
- resource unref/flush 顺序。

## 12. VFIO/vGPU 显示

### 12.1 VFIO

Guest 原生 GPU driver 可直接控制物理 display engine：

- Guest KMS modeset；
- physical connector/display assignment；
- framebuffer/scanout；
- cursor/plane。

Host 可能失去该 GPU/display。单 GPU Host 直通会影响本地控制台。

### 12.2 vendor vGPU

常见：

- Guest virtual GPU driver/KMS；
- Host vGPU manager/display channel；
- framebuffer capture/encode；
- remote client。

显示帧正确、cursor 正确、remote protocol 和输入转发是不同验收面。

## 13. VDI 路径

VDI = Guest 图形 + Host remote encoder：

```text
Guest app/compositor
  -> virtual display
  -> host capture/encode
  -> network
  -> client decode/present
```

关键因素：

- 帧捕获/readback；
- H.264/AV1 encoder；
- damage/region；
- cursor 独立 overlay；
- frame pacing；
- network congestion；
- client decode/render；
- 多显示器/分辨率。

GPU 渲染快不代表 VDI 延迟低。

## 14. Display/Scanout 同步

### 14.1 Render completion

producer 完成 render 后才能把 image 交给 compositor。

### 14.2 Acquire

consumer 获得 image 访问权，并等待 producer fence。

### 14.3 Release

consumer 用完 image 后 signal，producer 才能复用。

### 14.4 Flip/scanout

page flip 提交新 framebuffer；真正扫描 out 后可能需要 vblank/present fence。

### 14.5 Post-process/readback

如果 Host 回读、格式转换或远程编码，还需要新的同步边界。

## 15. 失败矩阵

| 症状 | 优先检查 |
| --- | --- |
| 窗口黑屏但离屏正确 | WSI format/modifier/compositor |
| cursor 正常但内容黑屏 | primary plane/scanout |
| 内容正确但大小错误 | rect/stride/offset/mode |
| resize 后旧内容 | display event/modeset/flush |
| dma-buf import 失败 | modifier/usage/fd type |
| 间歇旧帧 | acquire/release/plane fence |
| 视频播放裁切 | pitches/offsets/fourcc |
| 远端花屏 | encoder copy/format/network |
| 输入位置偏 | cursor hotspot/scaling/client |
| swapchain 创建成功但无画面 | acquire/render/present 全链 |

## 16. 性能指标

- frame submit latency；
- GPU render completion；
- compositor acquire/release；
- scanout flip latency；
- frame drops；
- readback bytes；
- CPU copy bandwidth；
- encoder latency/bitrate；
- network RTT/jitter；
- present P50/P99；
- multi-display overhead。

GPU utilization 不能代表 VDI 端到端延迟。

## 17. 安全检查

- resource ID/scanout range；
- rect/stride/offset overflow；
- dmabuf format/modifier/fd ownership；
- cursor resource size；
- image lifetime 到 present completion；
- Host 回读/trace 中的用户画面；
- multiple tenant display isolation；
- remote session token/input channel；
- confidential GPU 的 framebuffer path。

## 18. 检查题

1. framebuffer、plane、CRTC、connector 各负责什么？
2. `SET_SCANOUT` 与 `RESOURCE_FLUSH` 有何不同？
3. 为什么 render fence 完成不代表 screen 已显示？
4. dumb buffer 的 damage 为什么特殊？
5. VirGL texture scanout 与 blob dma-buf scanout 路径有何区别？
6. `wl_shm` 与 dma-buf WSI 的拷贝/同步差异是什么？
7. swapchain 创建成功证明了什么，不能证明什么？
8. DRM modifier 不匹配会在哪一层失败？
9. VFIO display 为什么可能让 Host 失去控制台？
10. VDI 中 GPU utilization 高为什么仍可能高延迟？
11. Host 回读为什么破坏 zero-copy 结论？
12. 为什么 cursor 正常不能证明 primary plane 正确？

## 19. 与其他专题

- GFXStream WSI：  
  [GFXStream 源码跟踪](../virtio-gpu/gfxstream-source-trace.md:268)
- Venus WSI：  
  [Venus 源码跟踪](../virtio-gpu/venus-source-trace.md:257)
- Native DRM：  
  [Native DRM 宏观](../virtio-gpu/native-drm-overview.md)
- 显存/dma-buf：  
  [GPU 显存与一致性](gpu-memory-semantics.md)
- dma-buf/fence/sync_file 源码：  
  [DMA-BUF 与 DMA-FENCE 源码跟踪](dma-buf-fence-source-trace.md)
- 执行/fence：  
  [GPU 执行模型](gpu-execution-model.md)
- 性能：  
  [GPU 性能与可观测性](gpu-performance-observability.md)
- VFIO/vGPU/MIG：  
  [平台方案比较](../virtio-gpu/platform-comparison.md)
