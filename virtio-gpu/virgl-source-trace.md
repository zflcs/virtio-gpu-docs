# VirGL 源码跟踪

## 1. 阅读范围

对应宏观文档：[virgl-overview.md](virgl-overview.md)。本路径对 Guest
`virtio_gpu` KMD 的复用方式见本文第 4 节；control queue、resource/blob、
execbuffer、virtqueue 和 fence completion 的公共逐函数分析见
[shared-kmd-source-trace.md](shared-kmd-source-trace.md)。

本文跟踪一条 OpenGL draw 从 Guest Mesa 到 Host GL 的源码路径。源码基线：

- Linux `6b4b543fa18757297ac756a6fe6e64f506b7844a`
- Mesa `ec7b2f322abbcd593d600a14afe2d2773512af4e`
- virglrenderer `e0a1caf3b147bb26dfe524dbb85ded1177e7e49e`
- QEMU `d73149283205645625bfa9fe029cfb68e4c4a9ae`

不把既有运行 trace 当作本次新实验；它们只用于解释事件。

宏观问题与本文源码入口的对应关系如下：

| 宏观问题 | VirGL 的源码答案 | 本文位置 |
| --- | --- | --- |
| Guest UMD 做什么 | Mesa VirGL Gallium driver 把 GL state/draw 编成 VirGL command | 第 3 节 |
| Guest KMD 做什么 | 复制 opaque command，引用 BO，构造 `SUBMIT_3D` 和 fence | 第 4 节、共用 KMD |
| 如何利用 Host GPU | vrend 解码后调用 Host OpenGL，由 Host GL driver 使用 GPU | 第 6–8 节 |
| 虚拟化形式 | Virtio 设备上的 OpenGL/Gallium API remoting，软件 parser 是边界 | 第 2、5–8 节 |
| 数据如何传输 | 命令经 control virtqueue/payload；resource 经 backing、transfer 和 blob | 第 3–5、9 节 |
| 控制逻辑 | capset/context/resource/attach/submit/flush 分属 UMD、KMD、QEMU、vrend | 第 2–9 节 |
| 内存与所有权 | Guest GEM 与 Host renderer resource 各有生命周期 | 第 9 节 |
| 同步与完成 | virtio fence 完成 Host renderer 工作，Guest 再 signal dma_fence | 第 10 节 |
| 呈现 | Guest KMS 选择 scanout，QEMU/display backend 呈现，渲染完成不等于可见 | 第 11 节 |

## 2. 总调用链

```text
glDraw*
  -> Mesa Gallium frontend
  -> virgl_draw_vbo()
  -> virgl_encoder_draw_vbo()
  -> VirGL command buffer
  -> virgl_flush_eq()
  -> virgl_drm_winsys_submit_cmd()
  -> DRM_IOCTL_VIRTGPU_EXECBUFFER
  -> virtio_gpu_execbuffer_ioctl()
  -> virtio_gpu_submit()
  -> virtio_gpu_cmd_submit()
  -> control virtqueue
  -> QEMU virtio_gpu_handle_ctrl()
  -> virtio_gpu_virgl_process_cmd()
  -> virgl_cmd_submit_3d()
  -> virgl_renderer_submit_cmd()
  -> vrend_decode_ctx_submit_cmd()
  -> vrend_decode_draw_vbo()
  -> vrend_draw_vbo()
  -> glDrawArrays/glDrawElements/glDraw*Indirect
```

这条链说明 VirGL 是 OpenGL/Gallium command remoting。Guest KMD 只搬运
opaque command stream，Host vrend 才理解 draw 语义。

## 3. Guest UMD：从 GL API 到命令流

### 3.1 Draw 前的隐式状态

[`virgl_draw_vbo()`](../../mesa/src/gallium/drivers/virgl/virgl_context.c)
在真正编码 `DRAW_VBO` 前处理索引/顶点资源，并调用：

```text
virgl_hw_set_index_buffer()
virgl_hw_set_vertex_buffers()
virgl_encoder_draw_vbo()
```

这说明 Guest UMD 不必为每个 GL state change 立即发一次内核 ioctl。很多状态
和资源绑定被编码到同一 command buffer，随后批量提交。

### 3.2 命令头

[`virgl_encoder_write_cmd_dword()`](../../mesa/src/gallium/drivers/virgl/virgl_encode.c:540)
从最高 16 bit 取得 payload dword 数：

```c
int len = dword >> 16;
if (ctx->cbuf->cdw + len + 1 > VIRGL_MAX_CMDBUF_DWORDS)
   ctx->base.flush(...);
```

所以 command buffer 逼近容量时，Mesa 会主动 flush。应用可见的 draw 次数和
Host 解码次数相同，但和 virtio 提交次数不一定相同。

### 3.3 Draw payload

[`virgl_encoder_draw_vbo()`](../../mesa/src/gallium/drivers/virgl/virgl_encode.c:985)
写入：

- opcode：`VIRGL_CCMD_DRAW_VBO`
- start/count
- primitive mode
- indexed
- instance count
- index bias/start instance
- primitive restart
- index bounds
- tessellation/drawid（可选）
- indirect buffer resource/offset/stride/count（可选）

Host 端必须按 variant 选择合法长度，不能只读固定前缀。

### 3.4 Resource 引用

[`virgl_encoder_emit_resource()`](../../mesa/src/gallium/drivers/virgl/virgl_encode.c:552)
把 Guest `virgl_resource` 的 winsys handle 写入 command stream。对应 winsys
还要把 handle 加入本轮 execbuffer 的 BO list，确保 KMD、Host 和 fence
生命周期覆盖命令引用。

### 3.5 Flush 与 fence

[`virgl_flush_eq()`](../../mesa/src/gallium/drivers/virgl/virgl_context.c:1086)：

1. 空 command buffer 且不要求 fence 时直接返回。
2. unmap pending upload。
3. 清 transfer queue，将待处理 upload 编进 command buffer。
4. 调 winsys `submit_cmd`。
5. 重置 sub-context 编码位置。

[`virgl_drm_winsys_submit_cmd()`](../../mesa/src/gallium/winsys/virgl/drm/virgl_drm_winsys.c:954)
填 `drm_virtgpu_execbuffer`：

```text
command / size
BO handle array
input fence fd
output fence fd, if requested
```

成功后，output fence fd 被封入 `virgl_drm_fence`，供后续
`fence_wait`/`fence_reference` 使用。

## 4. Guest KMD：execbuffer 顺序

入口
[`virtio_gpu_execbuffer_ioctl()`](../../linux/drivers/gpu/drm/virtio/virtgpu_submit.c:478)
的实际顺序：

1. 拒绝无 VirGL 3D feature 的设备。
2. 检查 flags 和 `ring_idx` 是否在 context 允许范围内。
3. `virtio_gpu_create_context()` 确保当前 `drm_file` 有 context。
4. `virtio_gpu_init_submit()` 分配 output fence。
5. 引用 BO list。
6. `vmemdup_user()` 复制用户 command bytes。
7. 建立 sync_file 和 output fence FD。
8. 解析 out deps/in syncobj。
9. 等待 foreign input fence。
10. lock BO reservation。
11. `virtio_gpu_submit()`。
12. 把 output fence FD 安装给用户。

重要设计点：等待 foreign fence 后才 lock BO list，减少持锁等待时间。Host
提交前锁定 BO，防止 resource backing 在命令引用期间被并发迁移/销毁。

### 4.1 内核提交与 virtqueue

[`virtio_gpu_submit()`](../../linux/drivers/gpu/drm/virtio/virtgpu_submit.c:353)
调 [`virtio_gpu_cmd_submit()`](../../linux/drivers/gpu/drm/virtio/virtgpu_vq.c:1345)。

后者设置：

```c
hdr.type = VIRTIO_GPU_CMD_SUBMIT_3D;
hdr.ctx_id = ctx_id;
cmd_p->size = data_size;
```

`data_buf/data_size` 指向内核复制出来的 VirGL stream，`objs` 保留 BO 引用。
`virtio_gpu_queue_fenced_ctrl_buffer()` 把 command header、payload、response
header 组织为 SG，并交给控制 virtqueue。

### 4.2 为什么 KMD 不理解 GL

KMD 没有 pattern match `DRAW_VBO`、`SET_BLEND` 或 shader opcode。它无法知道
command 中引用了哪些 GL 对象，只能依赖 UMD 提供的 BO handle list、context
和 fence。协议错误最终由 Host decoder 拒绝。

## 5. Virtio/QEMU 接收

QEMU 的 final dispatch 是
[`virtio_gpu_handle_ctrl()`](../../qemu/hw/display/virtio-gpu.c:1323)：

1. `virtqueue_pop()` 取 command。
2. 放入 `cmdq`。
3. [`virtio_gpu_process_cmdq()`](../../qemu/hw/display/virtio-gpu.c:1253)
   按 order 处理。
4. 如果 renderer blocked，停止处理。
5. 没有 fence 且未 finished 的 command 可 suspend。
6. 有 fence 的 command 转入 `fenceq`，等 Host completion。

`VIRTIO_GPU_CMD_SUBMIT_3D` 进入
[`virgl_cmd_submit_3d()`](../../qemu/hw/display/virtio-gpu-virgl.c:632)：

1. 检查 `cs.size` 不超过 `VIRTIO_GPU_MAX_CMD_SUBMIT_SIZE`。
2. `g_try_malloc()` 分配临时 buffer。
3. `iov_to_buf()` 从 Guest SG 复制 payload。
4. 校验实际 bytes 等于 `cs.size`。
5. 调 `virgl_renderer_submit_cmd(buf, ctx_id, size / 4)`。

这里能直接看到一次 Guest-to-QEMU payload copy。

## 6. virglrenderer 选择 context

[`virgl_renderer_context_create_with_flags()`](../../virglrenderer/src/virglrenderer.c:239)
从 context flags 取 capset ID：

- VirGL/VirGL2 -> `vrend_renderer_context_create()`
- Venus -> `proxy_context_create()`
- DRM -> `drm_renderer_create()`

context 建立后记录 `ctx_id/capset_id/fence_retire`，并加入全局 context
table。

[`virgl_renderer_submit_cmd()`](../../virglrenderer/src/virglrenderer.c:319)：

1. 按 `ctx_id` 查 context，不存在返回 `EINVAL`。
2. 校验 dword 数没有溢出。
3. 校验 command buffer 4-byte 对齐。
4. 调 `ctx->submit_cmd(ctx, buffer, ndw * 4)`。

## 7. vrend 解码 draw

[`vrend_decode_ctx_submit_cmd()`](../../virglrenderer/src/vrend/vrend_decode.c:2061)
循环读取 command header，用 decode table 找 handler。

对 draw：

[`vrend_decode_draw_vbo()`](../../virglrenderer/src/vrend/vrend_decode.c:463)
先严格匹配三种合法 payload size：

```text
VIRGL_DRAW_VBO_SIZE
VIRGL_DRAW_VBO_SIZE_TESS
VIRGL_DRAW_VBO_SIZE_INDIRECT
```

然后构造 `pipe_draw_info`，读取 indirect resource handle，调用
`vrend_draw_vbo()`。

## 8. Host GL 执行

[`vrend_draw_vbo()`](../../virglrenderer/src/vrend/vrend_renderer.c:5890)
不只是最后一行 `glDraw*`，还负责：

- feature capability 校验；
- indirect resource lookup；
- context switch completion；
- stencil/scissor/viewport/blend dirty state；
- shader/program 选择和 link；
- uniform 和 system value 更新；
- VAO/vertex element/index buffer 绑定；
- transform feedback；
- primitive restart；
- indirect parameter buffer；
- 最终 `glDrawArrays*`/`glDrawElements*`。

因此 Host renderer 的 CPU 成本可能远高于 Guest 编码成本，尤其在小 draw 和
频繁 pipeline/state 切换时。

## 9. Resource 与 transfer 源码

VirGL 的 resource 生命周期跨越：

```text
Guest GEM object
  -> hw_res_handle
  -> execbuffer BO list
  -> virgl_resource table
  -> vrend resource + Host GL id
```

主要操作：

- `RESOURCE_CREATE_3D`：创建 Guest 与 Host resource 映射。
- `RESOURCE_ATTACH_BACKING`：让 Host 知道 Guest storage 的 mem entry/iovec。
- `TRANSFER_TO_HOST_3D`：上传 texture/buffer 子区域。
- `TRANSFER_FROM_HOST_3D`：回读。
- `CTX_ATTACH_RESOURCE`/`DETACH_RESOURCE`：控制 context 对 resource 的引用。
- `RESOURCE_UNREF`：销毁 Host 对象并解除 Guest handle。

Host resource 不应仅依赖 Guest handle table 的生命周期；renderer 内部还要
保留自己的引用，直到 pending Host GL work 完成。

## 10. Fence 返回路径

```text
vrend Host GL completion
  -> context fence retire
  -> per_context_fence_retire()
  -> QEMU write_context_fence callback
  -> QEMU fence queue matching
  -> virtio_gpu_ctrl_response_nodata()
  -> Guest KMD fence completion
  -> dma_fence/sync_file
  -> Mesa fence_wait
```

QEMU 不能只处理 fence queue 头部，因为不同 context/ring 的 completion 可能
乱序。`VIRTIO_GPU_FLAG_INFO_RING_IDX` 将 completion 限定到
`ctx_id + ring_idx` timeline。

## 11. Display 路径

渲染 resource 到显示不是自动发生的：

1. Guest compositor/KMS 选择 scanout resource。
2. Guest 发 `SET_SCANOUT`。
3. 更新显示区域时发 `RESOURCE_FLUSH`。
4. QEMU 根据 VirGL resource texture、blob dmabuf 或常规 resource 更新
   console/scanout。
5. Host display backend 绘制。

VirGL renderer 的 texture 内容正确仍不能证明 scanout rect、format、host
texture id 和 display backend 正确。

## 12. 关键失败点

| 症状 | 优先源码检查 |
| --- | --- |
| capset 缺失 | QEMU GL properties / `virtio_gpu_virgl_get_capsets()` |
| create context 失败 | `virgl_renderer_context_create_with_flags()` |
| submit 返回错误 | QEMU size / iov copy / decoder length |
| Host GL 不执行 draw | `vrend_draw_vbo()` 的 shader/resource validation |
| fence 永不返回 | vrend fence retire callback 与 QEMU fenceq |
| resource 内容错误 | backing attach、transfer、Host GL texture |
| 黑屏但离屏正确 | `SET_SCANOUT`/`RESOURCE_FLUSH`/display path |

## 13. 推荐源码阅读顺序

1. `virgl_encode.c` 的 command header、state、draw。
2. `virgl_context.c` 的 draw/flush。
3. `virgl_drm_winsys.c` 的 execbuffer 组装。
4. `virtgpu_submit.c` 与 `virtgpu_vq.c`。
5. QEMU 的 control queue 和 VirGL adapter。
6. `virglrenderer.c` context table。
7. `vrend_decode.c` decode table。
8. `vrend_renderer.c` 的 draw 和 Host GL 调用。

## 14. 检查题

1. 为什么 Guest KMD 无法校验 draw payload 的语义？
2. 哪段代码决定一次 Mesa draw 是否触发新的 virtio 提交？
3. QEMU 的 payload copy 与 Guest KMD 的 `vmemdup_user()` 分别何时发生？
4. BO list 在提交和 fence 生命周期中解决什么问题？
5. 为什么 fence queue 必须按 context 和 ring 匹配？
6. `RESOURCE_FLUSH` 与 Host GL execution completion 有什么区别？
7. DRAW payload 的三个合法长度分别来自哪些 GL 场景？
8. Host GL 调用失败时，QEMU response 与 Guest 可见错误如何传播？
