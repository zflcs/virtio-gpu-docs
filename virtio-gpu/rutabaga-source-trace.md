# Rutabaga 源码跟踪

## 1. 阅读范围

对应宏观文档：[rutabaga-overview.md](rutabaga-overview.md)。Rutabaga 位于
Host/QEMU 侧，不替代 Guest KMD；它接收的 command 已经由 Guest
`virtio_gpu` KMD 放入 virtqueue。Guest KMD 的公共机制见
[shared-kmd-source-trace.md](shared-kmd-source-trace.md)。

本文重点是 Host/QEMU 侧：virtio-gpu command 如何穿过 QEMU 的 Rutabaga
adapter、C FFI、Rust core，进入所选 `RutabagaComponent`。GFXStream 的 Guest
encoder 和 Host decoder 细节见
[gfxstream-source-trace.md](gfxstream-source-trace.md)。

宏观问题与本文源码入口的对应关系如下：

| 宏观问题 | Rutabaga 的源码答案 | 本文位置 |
| --- | --- | --- |
| Guest UMD 做什么 | Rutabaga 没有自己的 Guest UMD，使用所选 component 对应的 UMD/协议 | 第 1、9 节 |
| Guest KMD 做什么 | 仍由 Guest `virtio_gpu` 运输 virtqueue command、resource 和 fence | 共用 KMD |
| Rutabaga 做什么 | QEMU 与 Rust component 之间的稳定 adapter/控制面 | 第 3–10 节 |
| 如何利用 Host GPU | `RutabagaComponent` 转发到 backend，当前重点是 GFXStream/Host Vulkan | 第 2、5、9 节 |
| 虚拟化形式 | VMM 进程内 component/API remoting，不是物理 PCI function 直通 | 第 2、3 节 |
| 数据如何传输 | Virtqueue → C FFI → mutable Rust slice → component/backend | 第 4–8 节 |
| 控制逻辑 | QEMU opcode dispatch、Rust trait method、component callback 三段式 | 第 4–9 节 |
| 内存与所有权 | QEMU Resource/MemoryRegion、Rust resource table、component object 分别拥有状态 | 第 7、9 节 |
| 同步与完成 | backend thread 的 fence callback 经 QEMU AIO BH 回到 virtio response | 第 8 节 |
| 生命周期边界 | 存在 snapshot FFI，但 QEMU device 仍有 migration blocker | 第 11 节 |

## 2. 总调用链

```text
virtio_gpu_rutabaga_handle_ctrl()
  -> virtio_gpu_rutabaga_process_cmd()
  -> rutabaga_cmd_*()
  -> librutabaga_gfx_ffi C ABI
  -> rutabaga_gfx::Rutabaga
  -> RutabagaContext method
  -> RutabagaComponent method
  -> Gfxstream
  -> stream_renderer_* (libgfxstream_backend)
  -> Host graphics API
```

## 3. QEMU device 初始化

### 3.1 Realize

[`virtio_gpu_rutabaga_realize()`](../../qemu/hw/display/virtio-gpu-rutabaga.c:1097)：

1. 拒绝 big-endian Host。
2. 注册 migration blocker：

   ```text
   rutabaga is not yet migratable
   ```

3. `virtio_gpu_rutabaga_init()` 初始化 Rust/Ffi backend。
4. 从 Rutabaga 获取 capset 数量。
5. 打开 Rutabaga/blob/context-init virtio features。
6. 设置 virtio config 的 capset 数。
7. 调通用 virtio-gpu realize。

### 3.2 Builder

[`virtio_gpu_rutabaga_init()`](../../qemu/hw/display/virtio-gpu-rutabaga.c:977)
构造 `struct rutabaga_builder`：

- `wsi=surfaceless|headless`
- `fence_cb=virtio_gpu_rutabaga_fence_cb`
- `debug_cb=virtio_gpu_rutabaga_debug_cb`
- `capset_mask`
- `user_data=(uint64_t)qemu_gpu`
- 可选 Wayland channel（cross-domain）

然后 `rutabaga_init()` 把 C builder 转成 Rust builder。

### 3.3 FFI 初始化

`rutabaga_init()`
（`rutabaga_gfx_ffi-0.1.3/src/lib.rs:200`）：

1. `catch_unwind` 包住整个初始化。
2. 把 C fence/debug functions 封装成 Rust handlers。
3. 把 C channel/path 转成 Rust `RutabagaChannel`。
4. 根据 capset mask 选择 component type。
5. 构造 `RutabagaBuilder`。
6. `Box::into_raw()` 返回 opaque Rutabaga pointer。

QEMU 永远不直接依赖 Rust struct layout，只持有 opaque pointer。

## 4. Control queue

[`virtio_gpu_rutabaga_handle_ctrl()`](../../qemu/hw/display/virtio-gpu-rutabaga.c:1076)
pop control virtqueue，插入 QEMU `cmdq`，再交通用
[`virtio_gpu_process_cmdq()`](../../qemu/hw/display/virtio-gpu.c:1253)
处理。

`process_cmd` 最终进入
[`virtio_gpu_rutabaga_process_cmd()`](../../qemu/hw/display/virtio-gpu-rutabaga.c:777)。
这是一个大型 opcode switch：

| virtio command | QEMU local handler | FFI/core target |
| --- | --- | --- |
| `CTX_CREATE/DESTROY` | `rutabaga_cmd_context_*` | `Rutabaga::create/destroy_context` |
| `RESOURCE_CREATE_2D/3D` | local resource create | component `create_3d` |
| `SUBMIT_3D` | `rutabaga_cmd_submit_3d` | `Rutabaga::submit_command` |
| `RESOURCE_CREATE_BLOB` | local + iovecs | `resource_create_blob` |
| `RESOURCE_MAP/UNMAP_BLOB` | QEMU hostmem subregion | `map/unmap` |
| `ATTACH/DETACH_BACKING` | QEMU iovec | component backing |
| `RESOURCE_FLUSH` | transfer read + display | component flush |
| `RESOURCE_UNREF` | QEMU cleanup + FFI | component unref |
| `GET_CAPSET_INFO/CAPSET` | QEMU response | component caps |

## 5. Submit 3D

[`rutabaga_cmd_submit_3d()`](../../qemu/hw/display/virtio-gpu-rutabaga.c:348)：

1. `VIRTIO_GPU_FILL_CMD` 取 command header。
2. 检查 `cs.size` 上限。
3. 分配临时 buffer。
4. `iov_to_buf()` 从 Guest SG 复制 command bytes。
5. 填 `struct rutabaga_command`。
6. 调 `rutabaga_submit_command()`。

FFI `rutabaga_submit_command()`
（`rutabaga_gfx_ffi-0.1.3/src/lib.rs:589`）把 raw pointer/length 转成
Rust slice，再调用 core。

Rust `Rutabaga::submit_command()`
（`rutabaga_gfx-0.1.3/src/rutabaga_core.rs:960`）：

1. 按 `ctx_id` 查 context。
2. 调 `RutabagaContext::submit_cmd()`。
3. component context 决定如何解码或转发 bytes。

当前 GFXStream component 的 `submit_cmd()` 只接受 `&mut [u8]`，拒绝非空
input fence，并调用 `stream_renderer_submit_cmd()`。

## 6. Context

### 6.1 QEMU -> FFI

QEMU [`rutabaga_cmd_context_create()`](../../qemu/hw/display/virtio-gpu-rutabaga.c:199)
把 `ctx_id/context_init/debug_name` 传给 `rutabaga_context_create()`。

### 6.2 FFI

`rutabaga_context_create()`
（`rutabaga_gfx_ffi-0.1.3/src/lib.rs:320`）：

- 校验 optional C string，转成 UTF-8 `Option<&str>`；
- 调用 `Rutabaga::create_context()`；
- 返回 C error code。

### 6.3 Core

`Rutabaga::create_context()`
（`rutabaga_gfx-0.1.3/src/rutabaga_core.rs:886`）：

1. 从 `context_init` 的低 8 bit 提取 capset ID。
2. capset ID 映射到 component；未知值使用 default component。
3. 检查 context ID 未重复。
4. component 创建 `Box<dyn RutabagaContext>`。
5. 加入 core context table。

`RutabagaContext` trait 将 `submit_cmd`、attach/detach resource、
context-specific blob 和 fence 抽象成统一接口。

## 7. Resource

### 7.1 Create 2D/3D

QEMU将 virtio resource metadata 转成 `rutabaga_create_3d`。Rust core
`resource_create_3d()` 分派到 default component，component 返回
`RutabagaResource`，再插入 resource table。

### 7.2 Blob

[`rutabaga_cmd_resource_create_blob()`](../../qemu/hw/display/virtio-gpu-rutabaga.c:622)：

1. 为 Guest resource 建 QEMU `virtio_gpu_simple_resource`。
2. 如果 blob 不是纯 Host3D，解析 Guest `nr_entries` 为 iovecs。
3. 包装 `rutabaga_create_blob`。
4. 调 `rutabaga_resource_create_blob(..., &vecs, NULL)`。

FFI 将 C iovec list 转成 `Vec<RutabagaIovec>`。Rust core
`resource_create_blob()`
（`rutabaga_gfx-0.1.3/src/rutabaga_core.rs:699`）：

1. 拒绝重复 resource ID。
2. 选择 default component，必要时处理 cross-domain context。
3. 调 component `create_blob()` 或 context `context_create_blob()`。
4. 把返回的 `RutabagaResource` 插入 table。

### 7.3 Map/Unmap

QEMU
[`rutabaga_cmd_resource_map_blob()`](../../qemu/hw/display/virtio-gpu-rutabaga.c:670)：

1. `rutabaga_resource_map_info()` 取 cache/access info。
2. `rutabaga_resource_map()` 取 Host pointer + size。
3. 把 pointer 包装成 QEMU `MemoryRegion`，放入 hostmem window。
4. 返回 virtio `RESP_OK_MAP_INFO`。

Unmap 时先删除 QEMU MemoryRegion，再调用
`rutabaga_resource_unmap()`。源码注释记录了 host mapping 与 QEMU
MemoryRegion 之间潜在的 use-after-unmap 风险。

## 8. Fence 与线程边界

通用 process command 发现有 `VIRTIO_GPU_FLAG_FENCE` 后，填
`struct rutabaga_fence` 并调用 `rutabaga_create_fence()`。

Host component 完成时调用 FFI builder 注册的 `fence_cb`。QEMU
[`virtio_gpu_rutabaga_fence_cb()`](../../qemu/hw/display/virtio-gpu-rutabaga.c:925)
不能直接从任意 backend callback thread 操 QEMU RCU/response 逻辑，所以：

```text
backend callback thread
  -> allocate rutabaga_aio_data
  -> aio_bh_schedule_oneshot()
  -> QEMU AIO context
  -> virtio_gpu_rutabaga_aio_cb()
  -> match ctx_id/ring_idx/fence_id
  -> virtio response
```

这是源代码层面的线程 contract，不只是性能实现。

## 9. Rust Component 接口

`rutabaga_core.rs` 定义两个核心 trait：

### 9.1 `RutabagaComponent`

负责：

- capset info/fill；
- create/unref resource；
- attach/detach backing；
- transfer read/write；
- blob create/export/map/unmap；
- resource flush；
- context create；
- component default fence。

### 9.2 `RutabagaContext`

负责：

- context-specific blob；
- `submit_cmd`；
- attach/detach resource；
- per-context/ring fence；
- component type。

GFXStream 实现把上述调用转发给 `stream_renderer_*` C symbols。Rutabaga 的
抽象价值在于 QEMU 不需要直接区分每个 Host library 的 ABI。

## 10. FFI 安全边界

FFI 源码大量使用：

- `#[repr(C)]`
- `from_raw_parts/from_raw_parts_mut`
- `Box::into_raw/from_raw`
- `catch_unwind(AssertUnwindSafe(...))`
- `-EINVAL`/`-ESRCH` 错误码

设计目的：

1. Rust panic 不穿过 C ABI。
2. C 只持有 opaque pointer。
3. Guest/QEMU 数据以显式 pointer + length 进入 Rust。
4. Rust-owned descriptor/handle 明确转移 owner。

但这些 unsafe block 假定 QEMU 传入合法 pointer 和长度。QEMU 侧仍需先校验
virtqueue、iovec、resource ID 和 command size。

## 11. Snapshot 与 migration

完整的 GPU VM 生命周期、VFIO migration、KFD CRIU 和 AI checkpoint 对照见
[GPU VM 生命周期、热插拔、Checkpoint 与迁移](../ai-gpu/vm-lifecycle-migration.md)。

Crate 中存在 `rutabaga_snapshot()`/`rutabaga_restore()` FFI 和部分 component
snapshot code，但当前 QEMU device realize 明确注册 migration blocker。
原因是完整迁移不仅需要 Rust state：

- QEMU virtio device state；
- Guest-visible resource/context；
- component/Host API object；
- mapping、fd 和 fence；
- Host/destination driver compatibility。

因此“Rust core 有 snapshot 函数”不等于 live migration 可用。

## 12. 关键失败点

| 症状 | 源码位置 |
| --- | --- |
| QEMU 启动 loading 失败 | `rutabaga_init()` / dynamic library |
| capset 数不对 | `rutabaga_get_num_capsets()` / component builder |
| context create 失败 | capset-to-component mapping |
| submit 失败 | QEMU size/copy、FFI slice、component submit |
| map 失败 | component map_info/map、QEMU MemoryRegion |
| fence 不返回 | component callback、AIO BH、ctx/ring matching |
| 无法迁移 | QEMU migration blocker 与未保存 Host state |

## 13. 推荐源码阅读顺序

1. `qemu/hw/display/virtio-gpu-rutabaga.c` 的 realize/init。
2. `rutabaga_gfx_ffi-0.1.3/src/lib.rs` builder 和 command entrypoints。
3. `rutabaga_gfx-0.1.3/src/rutabaga_core.rs` trait/table。
4. `rutabaga_gfx-0.1.3/src/gfxstream.rs` component 实现。
5. QEMU process command 和 fence callback。
6. cross-domain/2D component，比较抽象是否真正通用。

## 14. 检查题

1. QEMU 的 Rutabaga device 如何处理 capset 选择？
2. Rust component 与 Guest UMD 的对应关系是什么？
3. FFI 为什么使用 opaque pointer 而不是暴露 Rust struct？
4. `catch_unwind` 能防住哪些错误，不能防住哪些错误？
5. `resource_create_blob` 的 iovec 所有权如何跨 C/Rust 边界？
6. QEMU MemoryRegion 与 component mapping 的释放顺序是什么？
7. fence callback 为何需要 `aio_bh_schedule_oneshot()`？
8. GFXStream component 如何实现 `RutabagaContext`？
9. 为什么有 snapshot API 仍不能直接 live migrate QEMU device？
10. 若换一个 component，哪些 Guest code 保持不变，哪些协议会变化？
