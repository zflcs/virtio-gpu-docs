# Vhost-user-gpu 源码跟踪：外置 virtio-gpu 后端

## 1. 定位

本文跟踪固定源码中的 QEMU `vhost-user-gpu`：

```text
QEMU front end (vhost-user-gpu-pci)
  <-> vhost-user virtqueue transport
  <-> external vhost-user-gpu backend process
  -> Pixman 2D or VirGL renderer
```

vhost-user-gpu 不是新的 Guest API，也不是 Venus/GFXStream capset。它是
virtio-gpu 的一种**进程边界和部署模式**：

- Guest 仍看到 virtio-gpu；
- Guest UMD/KMD 不变；
- QEMU 不再直接处理 3D command；
- 外置后端直接消费 virtqueue，并调用 renderer；
- 外置后端通过独立 display protocol 把 cursor/scanout/update 发回 QEMU。

固定源码：

```text
QEMU d73149283205645625bfa9fe029cfb68e4c4a9ae
qemu/hw/display/vhost-user-gpu.c
qemu/hw/display/vhost-user-gpu-pci.c
qemu/contrib/vhost-user-gpu/
qemu/docs/interop/vhost-user-gpu.rst
```

本文不启动 QEMU、外置 GPU 进程、VM 或 renderer 实验。

## 2. 为什么不等于四条路径之外的第五条 API 路径

对照：

| 路径 | 是否改变 Guest API | Host renderer |
| --- | --- | --- |
| VirGL in QEMU | OpenGL/Gallium remoting | QEMU 进程内 vrend |
| Venus | Vulkan remoting | proxy/vkr |
| GFXStream/Rutabaga | API streaming | GFXStream backend |
| vhost-user-gpu | 不改变；当前 contrib 后端主要 2D/VirGL | 外置进程 |

当前固定版本的 `contrib/vhost-user-gpu` 只显式提供：

- 2D/Pixman；
- `VIRTIO_GPU_F_VIRGL`；
- VirGL/VirGL2 capset；
- EDID；
- cursor/scanout；
- dma-buf scanout/update。

源码没有在该 contrib backend 中注册 Venus、GFXStream、Rutabaga、resource
blob 或 native DRM context。部署某个发行版扩展后端时必须单独验证。

## 3. 两个 socket/protocol 不要混淆

### 3.1 vhost-user virtqueue socket

QEMU 与外置后端之间首先建立 vhost-user 连接，用于：

- feature/queue 协议；
- Guest memory table；
- virtqueue kick/call；
- config；
- 后端启动/停止。

后端通过 `libvhost-user` 直接 pop/push Guest virtqueue。

### 3.2 vhost-user-gpu display protocol

另一个由 `VHOST_USER_GPU_SET_SOCKET` 传入的 UNIX stream socket 专门传：

- display info；
- cursor；
- scanout；
- dirty update；
- EDID；
- dma-buf fd；
- dma-buf modifier。

消息方向与普通 display server 相反：

```text
backend (client) -> QEMU (display server)
  scanout/cursor/update

QEMU -> backend
  optional reply
```

协议文档明确说请求由 backend 发出，可选 reply 由 frontend 返回。

## 4. QEMU 前端初始化

[`vhost_user_gpu_device_realize()`](../../qemu/hw/display/vhost-user-gpu.c:659)
初始化 2 个 vhost virtqueue，并映射 feature：

- backend 有 `VIRTIO_GPU_F_VIRGL` -> QEMU 打开 VirGL；
- backend 有 `EDID` -> QEMU 打开 EDID；
- backend 有 `RESOURCE_UUID` -> QEMU 打开 UUID capability；
- vhost-user-gpu 默认声明 dma-buf display capability。

2 个 queue 对应：

- control queue；
- cursor queue。

这与常规 virtio-gpu control/cursor queue 一致，只是由外置后端处理。

### 4.1 DRIVER_OK 后启动

[`vhost_user_gpu_set_status()`](../../qemu/hw/display/vhost-user-gpu.c:557)
只有在 Guest 设置 `VIRTIO_CONFIG_S_DRIVER_OK` 且 VM running 时：

1. 建立 UNIX socketpair；
2. 把 display protocol fd 作为 vhost-user message 发给 backend；
3. 启动 vhost backend。

暂停/reset 时停止 backend 并解除 socket handler，避免后台继续访问 Guest
virtqueue。

### 4.2 display channel 阻塞

[`vhost_user_gpu_update_blocked()`](../../qemu/hw/display/vhost-user-gpu.c:449)
可暂停/恢复读取 display channel。

[`vhost_user_gpu_gl_flushed()`](../../qemu/hw/display/vhost-user-gpu.c:456)
在 renderer/display flush 完成后解除 backend blocked 状态。这个回调表达：

```text
backend 请求 display update
  -> QEMU/display backend 尚未可继续
  -> 暂缓 backend 消息
  -> flush 完成后 resume
```

不能把它误认为 Guest GPU fence 完成。

## 5. 外置 backend 初始化

`contrib/vhost-user-gpu`
[`main()`](../../qemu/contrib/vhost-user-gpu/vhost-user-gpu.c:1250)：

1. 解析 `--fd`/`--socket-path`、`--render-node`、`--virgl`；
2. 打开 render node；
3. 初始化 vugbm；
4. 建立/接收 UNIX socket；
5. `vug_init()` 初始化 libvhost-user；
6. 进入 GLib main loop。

如果 `--virgl` 未开启，backend 支持 2D path；开启后初始化 virglrenderer。

### 5.1 Feature

[`vg_set_features()`](../../qemu/contrib/vhost-user-gpu/vhost-user-gpu.c:1160)
根据 negotiated `VIRTIO_GPU_F_VIRGL` 初始化 renderer。

[`vg_get_features()`](../../qemu/contrib/vhost-user-gpu/vhost-user-gpu.c:1147)
对 frontend 暴露 VirGL 和 EDID。

### 5.2 Queue handler

[`vg_queue_set_started()`](../../qemu/contrib/vhost-user-gpu/vhost-user-gpu.c:1056)：

- queue 0 -> `vg_handle_ctrl()`；
- queue 1 -> `vg_handle_cursor()`。

### 5.3 SET_SOCKET

[`vg_process_msg()`](../../qemu/contrib/vhost-user-gpu/vhost-user-gpu.c:1127)
处理 `VHOST_USER_GPU_SET_SOCKET`，保存 display fd，并开始协商：

- EDID protocol feature；
- dmabuf2/modifier protocol feature。

## 6. 控制队列处理

[`vg_handle_ctrl()`](../../qemu/contrib/vhost-user-gpu/vhost-user-gpu.c:903)：

1. 若正在等待 display reply，暂停消费；
2. `vu_queue_pop()` 取 command；
3. 读取并校验 `virtio_gpu_ctrl_hdr`；
4. endian 转换；
5. VirGL mode -> `vg_virgl_process_cmd()`；
6. 否则 -> 2D `vg_process_cmd()`；
7. 未完成 command加入 `fenceq`；
8. 完成 response 写回 in_sg，push queue 并 notify Guest。

### 6.1 Response 与 fence

[`vg_ctrl_response()`](../../qemu/contrib/vhost-user-gpu/vhost-user-gpu.c:230)：

- 如果 request 有 `VIRTIO_GPU_FLAG_FENCE`，response 复制 fence id/ctx id；
- 写 in_sg；
- push descriptor；
- queue notify。

Guest virtio fence 的 completion 在这一层完成。它与 display protocol 的
`DMABUF_UPDATE` reply 是两个不同事件。

### 6.2 同步等待 head-of-line

某些需要 frontend reply 的命令设置 `wait_in`，例如：

- `GET_DISPLAY_INFO`；
- `GET_EDID`；
- `DMABUF_UPDATE`。

`vg_handle_ctrl()` 发现 `wait_in != 0` 就停止继续 pop。此时一个长等待 display
reply 会阻塞后续 control command。这是外置协议带来的 head-of-line 风险。

## 7. 2D/Pixman 数据面

### 7.1 Resource create

[`vg_resource_create_2d()`](../../qemu/contrib/vhost-user-gpu/vhost-user-gpu.c:353)
检查 resource ID、format，创建 vugbm buffer 和 Pixman image。

### 7.2 Attach backing

[`vg_create_mapping_iov()`](../../qemu/contrib/vhost-user-gpu/vhost-user-gpu.c:477)
：

- 限制 `nr_entries <= 16384`；
- 读取 Guest mem entries；
- 用 vhost-user shared memory `vu_gpa_to_va()` 转 Host VA；
- 校验 GPA 映射长度。

外置进程能访问 Guest RAM 依赖 vhost-user memory region 配置。

### 7.3 Transfer

`vg_transfer_to_host_2d()` 从 Guest iovec 拷贝到 Pixman buffer。当前 path 有
CPU copy，不是“外置进程自动零拷贝”。

### 7.4 Scanout/update

[`vg_set_scanout()`](../../qemu/contrib/vhost-user-gpu/vhost-user-gpu.c:654)：

- 有 dma-buf -> `VHOST_USER_GPU_DMABUF_SCANOUT` + SCM_RIGHTS fd；
- 无 dma-buf -> `VHOST_USER_GPU_SCANOUT` 元数据。

[`vg_resource_flush()`](../../qemu/contrib/vhost-user-gpu/vhost-user-gpu.c:746)：

- dma-buf -> dirty rect `DMABUF_UPDATE`，然后等待 reply；
- shm/Pixman -> 复制 dirty pixels 到 `VHOST_USER_GPU_UPDATE` 消息。

## 8. VirGL 外置数据面

### 8.1 Capset

`virgl_cmd_get_capset_info()`返回 VirGL 和 VirGL2 两项。backend VirGL feature
未初始化时不会声明 3D path。

### 8.2 Resource attach

[`virgl_resource_attach_backing()`](../../qemu/contrib/vhost-user-gpu/virgl.c:307)
将 Guest iovec 交给：

```text
virgl_renderer_resource_attach_iov()
```

与 QEMU 进程内 VirGL adapter 的思路相同，但调用者变成外置进程。

### 8.3 Submit

[`virgl_cmd_submit_3d()`](../../qemu/contrib/vhost-user-gpu/virgl.c:208)：

1. 检查 command size；
2. 从 Guest iovec 复制 payload；
3. 调 `virgl_renderer_submit_cmd()`。

这里仍有 Guest-to-backend copy。Host renderer 后续可能继续解码并执行 GL。

### 8.4 Scanout

[`virgl_cmd_set_scanout()`](../../qemu/contrib/vhost-user-gpu/virgl.c:380)：

- `virgl_renderer_resource_get_info()`/`_ext()` 取 texture；
- `virgl_renderer_get_fd_for_texture()` 导出 dma-buf；
- 通过 display socket 发送 fd；
- 有 dmabuf2 negotiated 时附 modifier。

[`virgl_cmd_resource_flush()`](../../qemu/contrib/vhost-user-gpu/virgl.c:457)
先 `glFlush()`，再发 `DMABUF_UPDATE`，并等待 QEMU reply/display flush。

`glFlush()` 是 Host GL 提交流程的一部分，不自动等于 GPU execution complete。

## 9. Cursor 队列

[`vg_handle_cursor()`](../../qemu/contrib/vhost-user-gpu/vhost-user-gpu.c:1018)
独立消费 queue 1：

- MOVE -> cursor position/hide；
- UPDATE -> 64x64 RGBA cursor data；
- 每次 pop 后 push空 response 并 notify。

cursor 与 control queue 分离，避免鼠标移动与大 resource/fence command
互相排队。

## 10. Display protocol wire contract

消息格式：

```text
u32 request
u32 flags
u32 size
payload
```

前端先检查 minimum payload size，再按 request 分派：

- `GET_DISPLAY_INFO`；
- `CURSOR_POS/POS_HIDE/UPDATE`；
- `SCANOUT`；
- `UPDATE`；
- `DMABUF_SCANOUT`；
- `DMABUF_UPDATE`；
- `GET_EDID`；
- `DMABUF_SCANOUT2`。

### 10.1 SCM_RIGHTS

dma-buf fd 通过 UNIX socket ancillary data 传递。

跨进程 fd 传递只共享内核 dma-buf object，不共享 userspace pointer。任一端
关闭自己的 fd 都不应提前销毁另一端的有效引用，但 producer/consumer fence
仍必须由协议或 dma-buf implicit/explicit sync 保证。

### 10.2 Modifier

`DMABUF_SCANOUT2` 增加 u64 modifier。若只协商 v1 scanout，display backend 可能
无法判断 tiled/compressed surface 的布局。

## 11. 进程边界与安全

### 11.1 收益

外置后端可以：

- 与 QEMU 主进程分离部署；
- 独立重启/crash containment；
- 独立 seccomp/namespace/sandbox；
- 独立 renderer library 和 GPU environment；
- 独立 profiling。

### 11.2 边界没有自动变小

vhost-user backend 仍需要访问：

- Guest RAM mapping；
- GEM/dma-buf；
- render node/GPU driver；
- display protocol；
- 第三方 virglrenderer/GL stack。

因此 vhost-user 本身不是安全沙箱。必须另外配置：

- 独立 Unix user/group；
- mount/network/device namespace；
- seccomp；
- 最小 device node 权限；
- socket path/ownership；
- dma-buf fd 权限。

### 11.3 parser 边界

2D/Pixman和 VirGL command parsing 都发生在 backend。QEMU 前端仍处理
vhost-user memory table/config，但 renderer parser crash 可以与 QEMU 主进程
分离，除非共享资源或 socket teardown 故障扩大影响。

## 12. 错误、阻塞与恢复

### 12.1 Backend 退出

外置进程异常会让 vhost backend unavailable。QEMU 可停止/重启 backend，但
Guest 的 in-flight command、resource table、renderer context 和 display
scanout 状态可能需要重建。

### 12.2 Socket 断开

`vg_send_msg()`/`vg_recv_msg()` 失败会关闭 display fd。backend 后续显示请求
无法送达，Guest 端未必立即看到错误。

### 12.3 Control queue 阻塞

等待 display reply 时 `wait_in` 阻止继续消费 control queue。一个未返回的
GET_EDID/DMABUF_UPDATE 可以让控制面停住。

### 12.4 Backend blocked

QEMU 可暂停读取 display channel；render flush 后恢复。若恢复信号丢失，可能
出现显示/控制路径停滞。

## 13. 生命周期

```text
QEMU realize
  -> vhost feature/queue setup
Guest DRIVER_OK
  -> create display socket
  -> VHOST_USER_GPU_SET_SOCKET
  -> backend start
Guest runtime
  -> ctrl/cursor virtqueues in backend
  -> display messages over socket
VM stop/reset
  -> frontend stops backend/socket handlers
backend exit
  -> destroy resources/renderer/socket
```

固定 contrib backend 的 resource table 位于外置进程内，而不是 QEMU。
因此迁移/CRIU/VM snapshot 不能只保存 QEMU device state；必须同时恢复 backend
resource、context、display 和 Host GPU object。本文没有验证任何 live migration
路径。

## 14. 与 QEMU 进程内 virtio-gpu 对照

| 维度 | in-process `virtio-gpu-gl` | `vhost-user-gpu` |
| --- | --- | --- |
| command consumer | QEMU | 外置 backend |
| parser/renderer library | QEMU 进程 | backend 进程 |
| Guest ABI | virtio-gpu | 同左 |
| Guest memory | QEMU AddressSpace | vhost-user shared memory |
| display | QEMU direct | backend -> QEMU protocol |
| fd sharing | QEMU 内部 | SCM_RIGHTS |
| crash domain | QEMU | 可分离 |
| extra latency | 低 | virtqueue + socket/display IPC |
| migration | 依 QEMU renderer | QEMU + backend 状态 |

## 15. 对 AI compute 的含义

当前固定 `vhost-user-gpu` contrib backend 主要是 2D/VirGL，不等于：

- Venus 外置化；
- GFXStream 外置化；
- ROCm compute proxy；
- CUDA/HIP API remoting；
- GPUDirect/RDMA；
- AI collective transport。

对 AI 的间接价值：

- 验证 renderer 进程隔离；
- 练习 dma-buf/fd/fence 跨进程；
- 评估 display/control round-trip；
- 为未来外置 API renderer 提供部署模型；
- 把 parser 崩溃与 VMM 主进程分离。

如果目标是 AI compute，不应只看 display socket；还要设计 command queue、
memory import、event/fence、topology 和 collective 的跨进程协议。

## 16. 失败矩阵

| 症状 | 优先检查 |
| --- | --- |
| Guest 无 virtio-gpu | QEMU device/vhost chardev/backend socket |
| Guest 设备存在但 3D 不可用 | backend `--virgl`、feature、render node |
| capset 数量不对 | backend `vg_virgl_get_num_capsets()` |
| command 无 response | vhost queue handler、backend wait_in、socket |
| 2D 内容错误 | attach backing/GPA mapping/Pixman copy |
| scanout 黑屏 | display protocol、dma-buf fd、fourcc/modifier |
| update 卡住 | DMABUF_UPDATE reply、QEMU display blocked/flushed |
| cursor 不显示 | cursor queue 或 display cursor message |
| backend 崩溃 | 外置进程日志、renderer、resource teardown |
| 迁移失败 | backend resource/context/fd/display state |

## 17. 推荐源码阅读顺序

1. `qemu/docs/interop/vhost-user-gpu.rst`。
2. QEMU `hw/display/vhost-user-gpu.c` 的 realize/status/display handler。
3. backend `main()`、feature 与 queue handler。
4. `vg_handle_ctrl()`、`vg_ctrl_response()`。
5. 2D resource/backing/scanout/update。
6. `contrib/vhost-user-gpu/virgl.c`。
7. `vugbm.c` 与 dma-buf。
8. 对照 `virtio-gpu-virgl.c` 和四路径源码文档。

## 18. 检查题

1. vhost-user-gpu 为什么不是新的 Guest API？
2. vhost-user socket 与 display protocol socket 各承载什么？
3. 为什么 display protocol 的请求方向是 backend -> QEMU？
4. QEMU 何时启动 backend，为什么不是 realize 时直接启动？
5. Guest command 如何到达外置进程，response 如何回到 Guest？
6. virtio fence 和 DMABUF_UPDATE reply 有什么区别？
7. `wait_in` 为什么可能阻塞后续 control command？
8. SCM_RIGHTS 传递 dma-buf 解决了什么，未解决什么？
9. 为什么 modifier 协议对 tiled/compressed surface 重要？
10. vhost-user 进程分离为什么不等于安全隔离？
11. 固定 contrib backend 支持哪些 capset，为什么不能宣称 Venus？
12. 外置 backend 的 resource table 如何影响 VM migration？
13. 如何测量 control socket、display IPC 和 Host renderer 各自延迟？
14. 外置 renderer 对 AI compute 需要额外补哪些协议？

## 19. 与其他专题

- [Virtio-GPU 四路径源码横向对照](source-trace-cross-path.md)
- [VirGL 源码跟踪](virgl-source-trace.md)
- [Venus 源码跟踪](venus-source-trace.md)
- [GFXStream 源码跟踪](gfxstream-source-trace.md)
- [GPU 虚拟化性能分析与可观测性](../ai-gpu/gpu-performance-observability.md)
- [GPU 虚拟化架构选型与评审 Playbook](../ai-gpu/gpu-virtualization-design-playbook.md)
