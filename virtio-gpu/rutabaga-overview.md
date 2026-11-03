# Virtio-GPU 路径四：Rutabaga

函数级源码链见 [rutabaga-source-trace.md](rutabaga-source-trace.md)。

## 1. 一句话定位

Rutabaga 是 Host/VMM 侧的 **虚拟图形接口（VGI）与组件适配层**。在 QEMU
中，它把 `virtio-gpu` 的 context/resource/submit/fence 命令翻译成后端组件
调用；当前实验主要使用 GFXStream component。Rutabaga 本身不是 Guest 图形
API driver，也不是独立 GPU renderer。

```text
Guest GFXStream (or another supported capset)
  -> Guest virtio-gpu DRM
  -> virtqueue
  -> QEMU virtio-gpu-rutabaga
  -> librutabaga_gfx_ffi C ABI
  -> Rust rutabaga_gfx core
  -> selected RutabagaComponent
       -> Gfxstream
       -> libgfxstream_backend
       -> Host graphics API/GPU
```

## 2. 为什么把 Rutabaga 单列

在源码和职责上要区分三件事：

| 名称 | 层级 | 主要职责 |
| --- | --- | --- |
| GFXStream | Host graphics backend | 解码 API stream，调用 Host graphics API |
| Rutabaga | VMM device backend framework | 抽象 context/resource/command，连接 component |
| QEMU Rutabaga | QEMU device adapter | virtio-gpu command 与 Rutabaga FFI 互转 |

把 GFXStream 直接等同于 Rutabaga 会遗漏 QEMU、Rust component trait、FFI
所有权和后端切换等关键边界。实际部署中两者经常一起出现，但并非同一层。

## 3. Guest UMD 和 KMD 如何工作

Rutabaga 只存在于 Host，因此 Guest 中没有“Rutabaga UMD”。Guest 仍使用
具体 capset 的 UMD：

- GFXStream Vulkan ICD；
- GFXStream GLES/composer（若启用）；
- 未来某个 Rutabaga component 对应的 capset。

Guest KMD 仍是 Linux virtio-gpu，负责：

- feature/capset 协商；
- context init；
- resource/blob create/unref；
- backing attach/detach；
- `SUBMIT_3D` 和 virtqueue；
- Guest fence/syncobj。

Rutabaga 的上线不会自动改变 Guest PCI topology：Guest 看到的仍是
`1af4:1050` virtio-gpu 设备，而不是 Host GPU 的 PCI function。

## 4. 如何使用 Host GPU

Rutabaga 不直接调用 GPU。它把命令分派给 `RutabagaComponent`，当前 GFXStream
component 通过动态库符号调用 `libgfxstream_backend`：

```text
QEMU
  -> Rutabaga FFI
  -> Gfxstream component
  -> stream_renderer_submit_cmd()
  -> GFXStream host backend
  -> Host Vulkan
  -> GPU
```

Host GPU 是否可共享、能支持哪些 API、需要哪些 external memory handle，均由
所选 component 和 Host driver 决定。Rutabaga 只是把设备控制面与具体 backend
解耦。

## 5. 这是什么形式的虚拟化

Rutabaga 属于：

- **VMM 侧虚拟设备后端框架**；
- 图形 API/组件 remoting 的 host adapter；
- 软件虚拟化；
- C ABI + Rust 核心的跨语言边界。

它不单独定义一种 GPU 隔离硬件，也不是 VFIO、SR-IOV 或 MIG。安全属性取决于：

- QEMU 进程权限；
- FFI 的指针/生命周期安全；
- 选择的 component；
- Host backend 的 parser 和 driver；
- Host GPU 的共享方式。

## 6. 控制逻辑

### 6.1 QEMU 初始化

QEMU 组装 `rutabaga_builder`：

- `capset_mask` 选择 GFXStream Vulkan、cross-domain 等能力；
- `wsi` 选择 surfaceless 等呈现模式；
- fence callback 把 Host completion 传回 QEMU；
- debug callback 收集错误；
- channels 为 display/cross-domain 提供 Host socket。

`rutabaga_init()` 创建 Rust `Rutabaga` 对象，并选择 component。

### 6.2 Command dispatch

QEMU 根据 virtio command 调不同 FFI 函数：

| QEMU 行为 | FFI/核心行为 |
| --- | --- |
| context create/destroy | `rutabaga_context_*` |
| resource create 2D/3D/blob | `rutabaga_resource_create_*` |
| submit 3D | `rutabaga_submit_command` |
| attach/detach backing | `rutabaga_resource_*_backing` |
| blob map/unmap | `rutabaga_resource_map*` |
| fence create | `rutabaga_create_fence` / context fence |

Rust core 管理 context ID、resource ID、component 和 fence handle，然后调用
所选 component 的 trait 方法。

### 6.3 Fence/completion

Host backend 完成 fence 后通过注册的 callback 回到 Rust core，再由
`rutabaga_fence_callback` 进入 QEMU。QEMU 再把结果转成 vitio-gpu 的 fence
response/event。必须保持：

- QEMU object 存活；
- Rust cookie 存活；
- component context/resource 存活；
- callback 不在已销毁 backend 上调用。

Rutabaga 的 FFI 使用 `catch_unwind`、错误码和显式 owner/handle 转换来避免
Rust panic 穿过 C ABI；但每个 unsafe 边界仍需逐一审查。

## 7. 数据如何传输

Rutabaga 有四种典型数据面：

1. **inline command**：QEMU 从 virtqueue 复制 `SUBMIT_3D` bytes，再传给
   component。
2. **iovec/backing**：Guest resource backing 的 scatter-gather memory。
3. **blob/handle/fd**：Host3D storage、mappable blob 或 dma-buf。
4. **fence callback**：完成事件从 backend 到 QEMU。

数据不是从 Guest 直接交给 GPU。路径至少是：

```text
Guest memory
  -> virtio
  -> QEMU
  -> Rust/FFI
  -> component
  -> Host API/driver
```

每一层都可能复制、映射或转换；Rutabaga 框架本身不承诺 zero-copy。

## 8. Resource 所有权

| 资源 | Guest 侧 | Rutabaga/QEMU 侧 | 释放顺序 |
| --- | --- | --- | --- |
| command bytes | UMD buffer | QEMU temporary/FFI buffer | submit 返回 |
| resource | GEM/resource ID | Rutabaga resource table | Guest unref / QEMU destroy |
| blob mapping | Guest mmap | Host backend mapping | unmap 后 unref |
| input fence | Guest sync fd | QEMU/Rutabaga fence IDs | wait 完成 |
| output fence | Guest dma_fence | backend callback | callback 后释放 |
| context | Guest DRM context | Rust component context | pending work 结束后 destroy |

尤其要在 reset、Guest teardown 和 backend device lost 场景中检查顺序。

## 9. 为什么有 Rust core 和 C FFI

QEMU 主体是 C，Rutabaga core 是 Rust。C FFI 提供稳定边界：

- `#[repr(C)]` 结构；
- 显式 raw pointer + length；
- Rust panic 不跨 C ABI；
- Rust 对象用 opaque pointer 管理；
- 错误转换为 C 整数；
- callback 使用 `extern "C"`。

这样 VMM 无需把整个 QEMU 改成 Rust，也能复用 Rust 组件和资源模型。代价是
ABI、所有权、线程和 panic 安全全在每个边界上人工维护。

## 10. 性能模型

```text
T = T_qemu_dispatch
  + T_virtio_copy
  + T_ffi
  + T_rutabaga_lookup
  + T_component_decode
  + T_host_api
  + T_gpu
  + T_fence_callback
  + T_present
```

Rutabaga 的抽象查找通常不是主要成本，真正成本多来自 command copy、
component decode、Host API validation、memory mapping 和显示。优化必须用
trace/profiling 拆分，不应只因为 Rust 边界就推断性能优劣。

## 11. 安全与生命周期

必须检查：

- C 指针是否可空、长度是否溢出；
- Guest 提供的 iovec、offset、resource ID 是否有上限；
- `catch_unwind` 是否覆盖所有可 panic 的路径；
- callback 能否在 object destroy 后触发；
- cross-thread 调用是否满足底层 API；
- component 动态库的路径、符号和 ABI；
- renderer/backend 权限是否被沙箱限制；
- migration blocker 是否允许 snapshot。

当前 QEMU Rutabaga 注册 `rutabaga is not yet migratable`，不能把“外围封装已
存在”视为迁移支持。

## 12. 优点、缺点与适用场景

优点：

- 把 virtio-gpu 通用命令与多个 graphics components 解耦；
- Rust core 可复用 memory/context/resource 基础设施；
- FFI 让 QEMU/C VMM 使用 Rust 组件；
- 支持 GFXStream 和 cross-domain 等方向；
- 为多 API/Android 图形虚拟化提供统一 adapter。

缺点：

- 多一层 FFI/component 抽象；
- Rust/C 生命周期边界复杂；
- 版本 ABI 和动态库 loading 容易出错；
- 当前 migration 不可用；
- 安全属性取决于 component，不能仅看 Rutabaga 名称。

适合 QEMU/Crosvm 等 VMM 承载 GFXStream 或 cross-domain 图形组件，尤其适合
Android/Cuttlefish 图形栈实验。

## 13. 关键源码入口

- QEMU 设备：
  `qemu/hw/display/virtio-gpu-rutabaga.c`
- FFI 定义和入口：
  `rutabaga_gfx_ffi-0.1.3/src/lib.rs`
- Rust core 组件接口：
  `rutabaga_gfx-0.1.3/src/lib.rs`
  `rutabaga_gfx-0.1.3/src/rutabaga_core.rs`
- GFXStream component：
  `rutabaga_gfx-0.1.3/src/gfxstream.rs`
- GFXStream backend 导出符号：
  `stream_renderer_*`，位于 `libgfxstream_backend`

## 14. 宏观检查题

1. Rutabaga 和 GFXStream 的责任边界是什么？
2. Guest 中为什么没有 Rutabaga UMD？
3. Rutabaga 如何选择 component 和 capset？
4. QEMU 的 virtio command 怎样进入 Rust trait？
5. FFI 为什么必须捕获 Rust panic？
6. resource/blob/fence 的所有权跨哪几个对象？
7. Rutabaga renderer 在实际部署中如何间接使用 Host GPU？
8. Rutabaga 是否提供 VFIO 式硬件隔离？
9. 为什么 Rutabaga 的 migration blocker 不能只靠 FFI 状态修复？
10. `librutabaga_gfx_ffi.so.0` 和 `libgfxstream_backend.so` 分别是什么层？
