# Virtio-GPU Native DRM Context 源码跟踪

## 1. 范围

对应宏观文档：[native-drm-overview.md](native-drm-overview.md)。Native
context 仍使用 Guest `virtio_gpu` KMD 的 context、blob、execbuffer 和
fence transport；公共实现见
[shared-kmd-source-trace.md](shared-kmd-source-trace.md)。

本文以 AMDGPU native context 为主线，跟踪：

```text
Guest amdgpu_virtio shim
  -> vdrm request/execbuffer
  -> virtio-gpu
  -> QEMU
  -> virglrenderer drm_context
  -> Host amdgpu renderer
  -> Host libdrm_amdgpu/amdgpu KMD
```

其他 native backend（MSM/i915/Asahi/Panfrost）共用 `vdrm`、QEMU 和通用
`drm_context`，本文只在通用层面说明。

源码基线：

- Linux `6b4b543fa18757297ac756a6fe6e64f506b7844a`
- Mesa `ec7b2f322abbcd593d600a14afe2d2773512af4e`
- virglrenderer `e0a1caf3b147bb26dfe524dbb85ded1177e7e49e`
- QEMU `d73149283205645625bfa9fe029cfb68e4c4a9ae`

本文不运行虚拟机、GPU workload 或实验。

## 2. 总体调用链

```text
Guest Mesa/vendor-like graphics UMD
  -> amdvgpu_* shim
  -> vdrm_send_req()/vdrm_execbuf()
  -> VIRTGPU_EXECBUFFER
  -> Linux virtio_gpu KMD
  -> virtio-gpu control virtqueue
  -> QEMU virgl_cmd_submit_3d()
  -> virgl_renderer_submit_cmd()
  -> drm_context_submit_cmd()
  -> AMDGPU CCMD handler
  -> Host libdrm_amdgpu
  -> Host amdgpu KMD
```

## 3. 初始化

### 3.1 Guest 打开 virtio render node

Mesa 的 `amdvgpu_device_initialize()` 打开 render node 后调用
`vdrm_device_connect(fd, VIRTGPU_DRM_CONTEXT_AMDGPU)`。

`vdrm_device_connect()` 根据 fd 选择 virtgpu transport：

- fd >= 0：`vdrm_virtgpu_connect()`；
- fd < 0：vtest pipe transport；
- 初始化 response 和 execbuffer lock。

### 3.2 Capset probe

`vdrm_virtgpu_connect()`：

1. `get_capset()` 调用 `VIRTGPU_GET_CAPS`；
2. 返回 `virgl_renderer_capset_drm`；
3. 检查 `caps.context_type == context_type`；
4. 不匹配时静默返回，避免多个 vdrm driver probe 时刷日志。

Capset 6 是协议入口。具体 Host GPU、driver UAPI 和 backend 能力还要看
capset 内容。

### 3.3 Context 初始化

`set_context()` 发送 `VIRTGPU_CONTEXT_INIT`：

```text
CAPSET_ID = VIRTGPU_DRM_CAPSET_DRM
NUM_RINGS  = 64
```

Guest KMD 建立 virtio-gpu context；Host renderer 之后按 capset/context type
创建 native context。

### 3.4 Control shmem

`init_shmem()`：

1. `VIRTGPU_RESOURCE_CREATE_BLOB`；
2. `blob_mem=HOST3D`；
3. `blob_flags=USE_MAPPABLE`；
4. `blob_id=0`；
5. `map_handle()` + mmap；
6. 读取 `shmem->rsp_mem_offset`，划分 response 区。

该 shmem 用于 request/response 和 Host processed seqno，不是主要 CS data
buffer。

## 4. Guest request 生成

### 4.1 Request 头

`vdrm_ccmd_req` 包含：

```text
cmd
len
seqno
rsp_off
```

每次 request 的 seqno 由 Guest transport 递增。

### 4.2 Response buffer

`vdrm_alloc_rsp()`：

1. 将请求 response size 按 8 byte align；
2. 在 shmem response 区环形分配；
3. 将 offset 写入 `req->rsp_off`；
4. 写入 `vdrm_ccmd_rsp.len`。

Guest 只分配“响应槽位”，Host 不应直接向共享内存中的任意地址写。

### 4.3 Buffering 和 flush

`enqueue_req()`：

- 写 seqno；
- 当前 request buffer 放不下时先 `flush_locked()`；
- 将 request 复制到本地 batch；
- 增加 request count。

非同步 request 可批量提交；同步 request 会 flush 并等待 fence 和 Host seqno。

### 4.4 Host processed seqno

`vdrm_host_sync()` 轮询共享 `shmem->seqno`，直到 Host 已处理 `req->seqno`。

该 seqno 只说明 Host 处理了 control request，不是 GPU completion。

## 5. Guest execbuffer

### 5.1 virtgpu backend

`virtgpu_execbuf_locked()` 填 `drm_virtgpu_execbuffer`：

- command pointer/size；
- BO handles；
- in fence FD；
- out fence FD；
- in/out syncobj；
- ring index。

随后调用 `DRM_IOCTL_VIRTGPU_EXECBUFFER`。

### 5.2 同步提交

`vdrm_send_req(..., sync=true)`：

1. 入队 request；
2. flush 获得 virtio out fence FD；
3. `sync_wait()` 等 FD；
4. `vdrm_host_sync()` 等 control seqno。

同步控件的顺序由 Guest transport 管理，不等同于所有 Host GPU work 的全局
同步。

## 6. AMDGPU shim

### 6.1 对象模型

Guest `amdvgpu_context` 维护：

- Host context ID；
- 每个 ring 的 next seqno；
- Guest vdrm device；
- context hash。

Guest BO 维护 virtio handle、resource ID、Host blob 和 Guest CPU mapping。

### 6.2 Query/info

`QUERY_INFO` 和 `BO_QUERY_INFO` 等同步 request 把 native query 参数编码到
control shmem，Host 返回硬件/对象信息。

### 6.3 Context create/destroy

`amdvgpu_context_create()`：

1. 在 Guest context hash 中查/管理本地 context；
2. 编码 `AMDGPU_CCMD_CREATE_CTX`；
3. 同步发送；
4. 读取 Host context ID；
5. 分配每 ring seqno 数组。

Destroy 使用同一 opcode 的 destroy flag，并从 Guest table 移除。

### 6.4 BO/GEM/VA

Guest object allocation 和 mapping 涉及：

- `GEM_NEW`；
- `BO_QUERY_INFO`；
- `BO_VA_OP`；
- `vdrm_bo_create()`；
- resource ID 查询；
- dma-buf import/export。

`vdrm_bo_create()` 会先 flush 之前 buffered commands，确保 Host 先看见旧
commands，再处理新 BO 创建。

### 6.5 CS submit 转换

`amdvgpu_cs_submit_raw2()` 先把 `drm_amdgpu_cs_chunk` 转成 vdrm CCMD：

1. 查 Guest context；
2. 遍历 native chunks；
3. 将 IB、dependencies、fence 数据复制到连续 payload；
4. 解析 BO list；
5. 将 AMDGPU syncobj chunks 转成 virtgpu syncobj；
6. 由 IB chunk 得到 virtio ring index；
7. 生成 descriptor table；
8. 编码 `AMDGPU_CCMD_CS_SUBMIT`；
9. 分配 response slot；
10. 调 `vdrm_execbuf()`；
11. 推进 Guest context 的 ring next seqno。

这一步是理解 native context 的核心：Guest 不是在传原始 `ioctl` struct，
而是在把 native object 语义重新编码成 vdrm/virtio transport。

### 6.6 Syncobj 细节

Guest shim 会把 AMDGPU syncobj chunks 转成 `drm_virtgpu_execbuffer_syncobj`，
因此它们由 virtio-gpu UAPI 处理，而不是继续作为 Host CCMD payload。

源码注释还指出 virtio-gpu 当前不支持 `WAIT_FOR_SUBMIT`，因此 shim 会清除
该 flag。这是明确的语义差异，不能声称与物理 amdgpu UAPI 完全等价。

## 7. Guest KMD/Virtio transport

Guest 仍走通用 `DRM_IOCTL_VIRTGPU_EXECBUFFER`：

- 引用 command 中的 BO/resource；
- 等待 input fences；
- 建立 output fence；
- 把 command bytes 发到 `VIRTIO_GPU_CMD_SUBMIT_3D`；
- 把 response/fence completion 回到 userspace。

Guest KMD 不解析 `AMDGPU_CCMD_CS_SUBMIT`。

## 8. QEMU 初始化

### 8.1 Feature/property

QEMU `virtio-gpu-gl` 提供 `drm_native_context` 属性。启用时需要：

- Host EGL；
- 足够新的 virglrenderer；
- `VIRGL_RENDERER_DRM` flag。

`virtio_gpu_virgl_init()` 在 `virtio_gpu_drm_enabled()` 时设置
`VIRGL_RENDERER_DRM`，再调用 `virgl_renderer_init()`。

### 8.2 Command 分发

Native context 复用 generic virtio-gpu command dispatch：

- context create/destroy；
- resource create/attach/map/unmap；
- submit；
- fence；
- scanout/flush。

`virgl_cmd_submit_3d()` 复制 Guest payload，再调用
`virgl_renderer_submit_cmd()`。

## 9. virglrenderer context selection

`virgl_renderer_context_create_with_flags()` 从 context flags 取 capset：

- VirGL -> vrend；
- Venus -> proxy；
- DRM capset -> `drm_renderer_create()`。

`drm_renderer_init()`：

1. 探测 Host render node；
2. 依次调用 native backend probe；
3. 选择匹配的 context type/backend；
4. 保存 Host DRM fd 和 capset。

AMDGPU backend probe 返回硬件信息、VA 上限、alignment 和 context type。

## 10. 通用 CCMD 解码

`drm_context_submit_cmd()` 是安全边界：

1. 检查总 size 不超过 `UINT32_MAX`；
2. 循环读取 `vdrm_ccmd_req`；
3. 检查 length 不小于 header；
4. 检查 length 不超过剩余 buffer；
5. 检查 request/response offset alignment；
6. 按 opcode 查 dispatch table；
7. 拒绝 trailing bytes。

### 10.1 Shadow response

`drm_context_rsp()`：

- 检查 response offset/size 是否在共享区内；
- 分配 shadow buffer；
- 清零，防止未初始化 Host heap 泄漏给 Guest；
- handler 完成后校验并复制；
- 发布 Host processed seqno。

源码明确说明 shared buffer 可被 Guest 写，直接使用会有 TOCTOU/竞态问题。

### 10.2 Double fetch

复制响应长度的代码用局部 `untrusted_len`，并警告不要把 volatile load inline
到 `MIN2()` 中，避免二次读取 Guest 可变长度。

这是 native command parser 的典型安全要求。

## 11. AMDGPU Host backend

### 11.1 Native context

`amdgpu_renderer_create()` 为 Guest context 建立：

- Host amdgpu device；
- Host amdgpu context；
- per-ring timeline；
- native object tables；
- fence retire callback。

### 11.2 CS submit

`amdgpu_ccmd_cs_submit()`：

1. 校验 request payload/descriptor/chunk offset/length；
2. 复制 Guest variable-length data，避免 handler 直接引用共享内存；
3. 查 context/object；
4. 构造 native chunk list；
5. 处理 BO handles、user fence、dependencies；
6. 导入 in fence/syncobj；
7. 调 Host `amdgpu_cs_submit_raw2()`；
8. 保存 native out fence FD；
9. 把 Guest ring/timeline 与 native fence 关联。

### 11.3 Resource import/export

AMDGPU renderer：

- Guest resource backing 可从 dma-buf export 后由 Host
  `amdgpu_bo_import()`；
- Host BO 可 export dma-buf；
- resource ID、blob ID、KMS handle 和 native BO 分别维护在不同表。

Resource 释放需要先确定 native BO 没有 pending access，再关闭 fd/unmap。

## 12. Fence 返回路径

```text
Host native amdgpu submit
  -> native out fence FD
  -> drm_timeline
  -> fence poll thread
  -> drm_context_fence_retire()
  -> virgl_context fence_retire
  -> QEMU fence callback
  -> virtio fence response
  -> Guest dma_fence/syncobj
```

AMDGPU `amdgpu_renderer_submit_fence()`：

- ring 0 的 Host CPU/sync completion 可直接 retire；
- 其他 ring 使用 native timeline；
- `drm_timeline_submit_fence()` 保存 native fd 并按 fence ID retire。

Control shmem seqno 不参与 GPU fence completion；它只表示 Host 处理 CCMD。

## 13. Resource 生命周期

### 13.1 Create

- Guest resource ID；
- Host `drm_object`；
- Host native BO；
- optional blob/dmabuf；
- resource table entry。

### 13.2 Map

- Guest map blob window；
- Host native BO CPU mapping/export；
- Guest VA 和 Host native VA 分别管理。

### 13.3 Detach/unref/context destroy

需要遵守：

1. 停止新 submit；
2. flush buffered control commands；
3. 等/取消 pending native work；
4. 撤销 Host object table；
5. unmap；
6. close native BO/fd；
7. 释放 Guest GEM/resource。

`virtgpu_bo_close()` 在发送 detach 前 flush buffered commands，就是为避免
资源销毁越过仍引用它的 command。

## 14. Reset

QEMU/VirGL reset 路径会：

- destroy resources；
- reset renderer；
- 清理 async context fences；
- 重置 scanout；
- 清理 command/fence queues。

Native backend 还必须恢复/重建 Host native device/context/timeline。Guest
driver 收到 device reset 后不能假设旧 native object/fence 仍有效。

## 15. Migration

当前 QEMU VirGL 设备注册 migration blocker。即便某个 native backend 有
snapshot 能力，也还需要：

- Guest virtio state；
- resources/backing；
- Host native BO；
- dma-buf/fd；
- GPU VA；
- native context/timeline；
- pending fences；
- Host GPU/driver 版本兼容。

因此不能把 native context 视为可透明 live migrate。

## 16. 失败定位

| 症状 | 优先检查 |
| --- | --- |
| capset 6 不存在 | QEMU property、EGL、virglrenderer version |
| context type 不匹配 | Guest shim 目标与 Host backend probe |
| control seqno 卡住 | Guest request 是否执行、QEMU/renderer submit |
| CS submit 失败 | chunk offset/length、BO handle、syncobj 转换 |
| resource 内容错误 | backing 导入、GPU VA、cache/flush |
| native fence 不返回 | Host submit fd、timeline retire、QEMU callback |
| unmap 卡住 | pending command/fence、host object 引用 |
| native backend 工作但窗口错误 | scanout/blob/flush/display |

## 17. 推荐源码阅读顺序

1. `mesa/src/amd/common/virtio/amdgpu_virtio_private.h`
2. `mesa/src/amd/common/virtio/amdgpu_virtio_proto.h`
3. `mesa/src/virtio/vdrm/vdrm.c`
4. `mesa/src/virtio/vdrm/vdrm_virtgpu.c`
5. `mesa/src/amd/common/virtio/amdgpu_virtio_bo.c`
6. `mesa/src/amd/common/virtio/amdgpu_virtio.c`
7. QEMU `virtio-gpu-gl.c`
8. virglrenderer `drm/drm_renderer.c`
9. virglrenderer `drm/drm_context.c`
10. virglrenderer `drm/amdgpu/amdgpu_renderer.c`
11. QEMU fence/response 和 scanout 路径

## 18. 检查题

1. Native context 为什么仍然要经过 `VIRTGPU_EXECBUFFER`？
2. Capset 6 的 context type 起什么作用？
3. 为什么 control shmem seqno 不是 GPU fence？
4. `vdrm_alloc_rsp()` 的 offset 如何被 Host 验证？
5. AMDGPU shim 如何处理 native chunks、BO list 和 syncobj？
6. 为什么 Guest handle/global resource ID/Host native BO 必须分开？
7. Host renderer 为什么使用 shadow response buffer？
8. `amdgpu_ccmd_cs_submit()` 做哪些输入校验？
9. ring 0 与其他 ring 的 fence 路径有什么差别？
10. 为什么 native resource unref 前需要 flush pending commands？
11. 当前 QEMU 为什么仍阻止 VirGL live migration？
12. Native DRM context 和 VFIO 的安全/隔离边界差在哪里？
