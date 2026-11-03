# Virtio-GPU 路径一：VirGL

函数级源码链见 [virgl-source-trace.md](virgl-source-trace.md)。

## 1. 一句话定位

VirGL 是面向 OpenGL/GLES 的 **API remoting** 路径：Guest Mesa 把 Gallium
图形操作编码成 VirGL command stream，经由 virtio-gpu 送到 Host，由
virglrenderer 的 vrend 解码并调用 Host OpenGL 驱动。

```text
Guest GL application
  -> Mesa OpenGL frontend / Gallium
  -> Mesa VirGL Gallium driver
  -> VirGL command buffer
  -> Guest DRM virtio-gpu
  -> virtqueue
  -> QEMU virtio-gpu
  -> virglrenderer / vrend
  -> EGL/GLX/GBM + Host OpenGL driver
  -> physical GPU
```

VirGL 关注的是“保持 OpenGL 应用和 Guest API 不变”，不是把物理 GPU
寄存器交给 Guest。

## 2. 系统层级

| 层 | virtio-gpu/VirGL 中的角色 |
| --- | --- |
| Guest 应用 | OpenGL/GLES 应用或合成器 |
| Guest UMD | Mesa GL frontend、Gallium、VirGL driver |
| Guest-visible kernel ABI | virtio-gpu DRM ioctl |
| Guest KMD | Linux `virtio_gpu` DRM driver |
| VM transport | virtio split virtqueue |
| VMM | QEMU `virtio-gpu-gl` |
| Host renderer | virglrenderer 的 `vrend` |
| Host API | OpenGL/EGL/GLX/GBM |
| Hardware | Host 物理 GPU |

## 3. Guest UMD 如何工作

应用仍调用标准 OpenGL entrypoint。Mesa OpenGL frontend 完成状态校验和
API-level 错误处理，Gallium 把调用转换成 `pipe_context` 操作。一旦选中
VirGL Gallium driver，工作不再是直接调用 Host GL：

1. 保存 set/blend/depth/rasterizer/sampler/shader 等状态。
2. 把 state 变化编码为 diff 或完整状态命令。
3. 把 draw、clear、copy、resource transfer 编码成 VirGL opcode。
4. 把 resource handle、shader handle、uniform buffer 等写入命令流。
5. command buffer 快满或调用 flush 时，把批次交给 winsys。

VirGL 的 UMD 同时承担“GL 兼容层”和“协议编码器”两个角色。Guest 看到的 GL
版本、extension 和 shader 能力受 Host GL、VirGL protocol 与 vrend 转换能力
共同限制。

## 4. Guest KMD 如何工作

Guest KMD 不理解 draw 或 GL state，它负责：

- 枚举 virtio-gpu feature；
- 向 Guest 用户态提供 DRM ioctl；
- 管理 GEM object、resource handle 和 drm_file context；
- 引用并锁定 submit 使用到的 BO；
- 处理 syncobj/foreign fence；
- 把用户 command buffer 复制到内核可提交内存；
- 组装 virtio control request、VirGL payload 和 response；
- kick control virtqueue；
- 接收 completion/fence response 并 signal Guest dma_fence。

对应用而言 `/dev/dri/renderD*` 是 virtio-gpu render node；它不暴露 NVIDIA
或 AMD 的真实 GPU ioctl。

## 5. 如何使用 Host GPU

QEMU 把 `SUBMIT_3D` 交给 virglrenderer。vrend 逐条解码 VirGL command，
重建 Host OpenGL 状态，最终执行 draw/dispatch/clear/copy 等 Host GL 调用。

Host GPU 的使用方式：

- Host GL driver 创建 texture、buffer、shader、program、framebuffer；
- 每个 Guest context 对应 vrend/Host API context；
- 多台 Guest 或一个 Guest 内多个 context 共享同一物理 GPU；
- Host 调度器和 GPU firmware 负责执行与时间片；
- Guest 无法直接访问 GPU MMIO、PCI config、MSI-X 或 Host GPU VA。

因此 Qualifier 是“共享 Host GPU 执行能力”，不是“VM 独占物理 GPU”。

## 6. 这属于什么形式的虚拟化

VirGL 同时包含两类机制：

- **split driver**：Guest 端客户端与 Host 端 renderer 配合；
- **API/command remoting**：转发高层 GL/Gallium 语义，而不是物理设备访问。

它是软件虚拟设备模型，不具备 VFIO 的 PCI function 所有权，也不具备
SR-IOV/MIG 的硬件 partition 隔离。Host renderer 是核心可信边界。

## 7. 控制逻辑

控制逻辑可以分为四段：

### 7.1 能力协商

QEMU 通告 3D、blob、context-init 等 feature 和 VirGL/VirGL2 capset。Guest
据此判断能否使用 VirGL，以及可用的 resource/blob 形式。

### 7.2 Context 建立

Guest `drm_file` 建立 VirGL context。Host 端建立 vrend context，准备 GL
状态、shader 缓存、resource table 和 fence 状态。

### 7.3 Resource 建立

`RESOURCE_CREATE_3D` 建立 Guest resource 和 Host GL resource 的映射。
Guest backing 或 Host3D blob 提供实际 storage。resource 还要按需 attach 到
context，才能被命令流引用。

### 7.4 Submit/Completion

Guest 执行 `EXECBUFFER`，KMD 发 `SUBMIT_3D`；QEMU 调 renderer；vrend 解码
并提交 Host GL work。Host GL fence 完成后，QEMU 通过 virtio fence response
让 Guest fence 完成。

## 8. 数据如何传输

### 8.1 Command stream

VirGL command stream 是主要数据面。命令头包含 opcode 和以 dword 计的长度，
payload 紧跟在后面。很多 GL 调用可以合并在一个 command buffer 中，因此：

```text
number of draw calls ≠ number of virtio submissions
```

### 8.2 Payload 路径

```text
Guest user command buffer
  -> copied by Guest KMD
  -> scatter-gather in virtqueue
  -> copied by QEMU to temporary buffer
  -> decoded by vrend
```

这是一条明确的 remoting 路径，不是 Guest 命令页直通物理 GPU。

### 8.3 Resource data

两种主要形式：

- Guest backing resource：Guest pages 通过 mem entry 提供，Host GL 在上传/
  读取时访问相应存储；
- blob resource：Host3D 分配，或 Host-visible/shared mapping，适合减少部分
  staging 和 transfer。

`TRANSFER_TO_HOST_3D`/`TRANSFER_FROM_HOST_3D` 仍然存在，尤其适合显式更新
texture/buffer 子区域。

### 8.4 显示数据

渲染完成不等于显示完成。resource 可能需要 flush；Guest KMS 合成后通过
virtio scanout 让 QEMU display 显示。Host renderer 的结果若要被 Guest
compositor 使用，还要经过 Guest-visible resource/dma-buf 路径。

## 9. 同步与生命周期

核心对象及生命周期：

| 对象 | 创建 | 使用 | 销毁 |
| --- | --- | --- | --- |
| Guest GEM resource | resource create | execbuffer/transfer | GEM close/unref |
| Host GL resource | resource create | attach/draw | unref response |
| vrend context | context create/init | SUBMIT_3D | context destroy |
| fence | submit output | wait/event | signal/drop |
| command buffer | UMD batch | execbuffer | submit complete |

必须同时考虑：

- Guest 用户 command buffer 生命周期；
- BO 在 submit 期间的 pin/reference；
- Host GL object 与 Guest resource handle 的引用；
- input/output fence；
- reset/unmap/teardown 时的 pending queue。

## 10. 性能模型

```text
T = T_guest_GL
  + T_encode
  + T_kernel_copy
  + T_virtqueue
  + T_qemu_copy
  + T_decode
  + T_host_GL_validation
  + T_gpu
  + T_sync
  + T_present
```

最容易被放大的 workload：

- 很多小 draw/state change；
- 高频 flush，导致 batch 变小；
- 频繁 resource create/attach/transfer；
- 频繁 fence wait；
- Host GL 对状态组合做慢路径校验或 shader recompile。

GPU-bound 的大三角形、重 shader、大分辨率 workload 中，协议成本占比可能
下降，但不能据此推断小 draw 或 compute-like dispatch 也接近原生。

## 11. 安全与隔离

需要重点审查：

- Guest 可构造的 command opcode、length 和 object handle；
- Host vrend 的边界检查、资源别名和越界访问；
- shader translation/compiler 的输入面；
- renderer 是否进程内运行、是否能访问超出 GPU 的 Host 资源；
- 多 Guest context 的资源隔离和 DoS 配额。

Host renderer 在独立进程中可缩小部署边界，但 VirGL 协议本身不是硬件隔离。

## 12. 优点、缺点与适用场景

优点：

- Guest API 标准，跨 NVIDIA/AMD/Intel Host 可移植；
- 开源栈完整，Linux/QEMU/Mesa/virglrenderer 可观测；
- 多个 Guest 可共享一张物理 GPU；
- 适合桌面、开发、CI、异构宿主。

缺点：

- OpenGL state 重建与命令解码成本；
- 功能取决于 Host GL 与 vrend；
- Host renderer 攻击面和版本耦合；
- display/WSI 仍可能成为独立瓶颈；
- 当前实验配置有 live migration blocker。

不适合把 VirGL 当作 VFIO 的替代品：它的优势是兼容和共享，不是硬件故障域
隔离或原生 PCI 语义。

## 13. 关键源码入口

- Guest GL 编码：
  `mesa/src/gallium/drivers/virgl/virgl_encode.c`
- Guest batch 提交：
  `mesa/src/gallium/drivers/virgl/virgl_context.c`
- DRM winsys：
  `mesa/src/gallium/winsys/virgl/drm/virgl_drm_winsys.c`
- Guest KMD submit：
  `linux/drivers/gpu/drm/virtio/virtgpu_submit.c`
- Guest virtqueue：
  `linux/drivers/gpu/drm/virtio/virtgpu_vq.c`
- QEMU command dispatch：
  `qemu/hw/display/virtio-gpu.c`
- QEMU VirGL adapter：
  `qemu/hw/display/virtio-gpu-virgl.c`
- virglrenderer context table：
  `virglrenderer/src/virglrenderer.c`
- VirGL decoder：
  `virglrenderer/src/vrend/vrend_decode.c`
- vrend Host GL execution：
  `virglrenderer/src/vrend/vrend_renderer.c`

## 14. 宏观检查题

1. VirGL 的 Guest UMD 和 KMD 分别知道哪些 GL 语义？
2. 为什么 VirGL 不需要 Guest 安装 Host GPU 的 GL driver？
3. Guest draw call 与 `SUBMIT_3D` 为什么不是一一对应？
4. Host renderer 如何知道 Guest 的 texture 和 buffer？
5. resource create、context attach、transfer 和 flush 分别解决什么？
6. Host GL fence 如何变成 Guest 可等待的 completion？
7. VirGL 与 VFIO 对 Guest-visible GPU 的根本差别是什么？
8. 哪些 workload 最可能放大 VirGL 的 CPU 和同步成本？
9. Host renderer 进程边界为什么不能单独证明安全隔离？
10. 如果 GL 应用画面错误，你会如何区分 Guest encoding、Host decode、
    Host GL 和 display scanout 四层问题？
