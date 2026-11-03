# Virtio-GPU 路径二：Venus

函数级源码链见 [venus-source-trace.md](venus-source-trace.md)。

## 1. 一句话定位

Venus 是面向 Vulkan 的 **API remoting** 路径：Guest Mesa Venus ICD 把
Vulkan 调用编码到共享 ring，Host virglrenderer 通过 proxy/render-server
交给 `vkr`，最终调用 Host Vulkan loader 和厂商 ICD。

```text
Guest Vulkan application
  -> Vulkan loader
  -> Mesa virtio ICD / Venus client
  -> Venus protocol in shared ring/blob
  -> Guest DRM virtio-gpu
  -> virtqueue
  -> QEMU
  -> virglrenderer proxy
  -> virgl_render_server
  -> vkr
  -> Host Vulkan
  -> physical GPU
```

Venus 不是 Vulkan 驱动的“GPU 寄存器直通”，也不是把 Host Vulkan handle
直接暴露给 Guest。Guest 和 Host 各自拥有 handle table 和对象生命周期。

## 2. 系统层级

| 层 | Venus 中的角色 |
| --- | --- |
| Guest 应用 | 标准 Vulkan 应用 |
| Loader | 按 ICD JSON 选择 `libvulkan_virtio` |
| Guest UMD | Mesa Venus client |
| Guest-visible kernel ABI | virtio-gpu DRM ioctl |
| Guest KMD | Linux virtio-gpu |
| Data plane | shared ring、descriptor、blob、dma-buf |
| VMM | QEMU |
| Host shim | virglrenderer Venus proxy |
| Host worker | optional/explicit `virgl_render_server` process |
| Host renderer | `vkr` Vulkan proxy |
| Host API | Vulkan loader + physical GPU ICD |

## 3. Guest UMD 如何工作

应用执行标准 Vulkan API。loader 选中 Mesa virtio ICD 后：

1. `vkCreateInstance` 初始化 Venus renderer 连接、协议版本和 capability。
2. Venus 创建共享 ring blob，映射 Guest 可见的 head/tail/status/command 区。
3. `vkCreateDevice`、queue、command pool、descriptor 等对象记录为 Guest
   object ID，并编码创建命令。
4. `vkCmd*` 写入 command buffer stream，不一定立即到达 Host。
5. `vkQueueSubmit` 把 command stream 放入 ring 或提交 batch，并推进 tail。
6. Guest 从共享 blob 取得 Host completion/ring seqno；Vulkan fence、
   semaphore、timeline 还要经过 Venus sync 映射。

Venus UMD 要对齐 Vulkan 的对象语义、thread safety、queue family、pipeline
layout、descriptor binding 和显式同步。它比 OpenGL remoting 编码状态更
明确，但协议也更复杂。

## 4. Guest KMD 如何工作

Guest KMD 不解析 Vulkan command。它：

- 通过 `CONTEXT_INIT` 接收 capset、ring 数量和 poll mask；
- 创建/管理 GEM blob 和 host-visible mapping；
- 把 `SUBMIT_3D` 控制命令放入 control virtqueue；
- 处理 resource attach/detach 和 fence；
- 在 Guest 用户态 mmap 共享 blob 时建立 Host-visible窗口；
- 对 `ring_idx` 对应 timeline 回写 fence completion。

Guest kernel 的职责是 transport、resource 和 synchronization，而不是 Vulkan
semantic implementation。

## 5. 如何使用 Host GPU

Host 侧有两条内部实现形式：

```text
QEMU -> libvirglrenderer proxy client -> render server process
                                        -> vkr -> Host Vulkan -> GPU
```

`vkr` 把 Guest object ID 映射到 Host Vulkan object。Host Vulkan 执行实际
image/buffer memory allocation、pipeline/shader compile、queue submit、
semaphore/fence 和 device lost 处理。

多 Guest 可以共享 Host Vulkan loader/ICD 和物理 GPU。Host GPU 通过 Vulkan
context/queue/device memory 被复用，Guest 不直接管理 GPU BAR、PCIe
function 或厂商 kernel ioctl。

## 6. 这属于什么形式的虚拟化

Venus 是：

- **Vulkan API remoting**；
- **split graphics driver**；
- 共享内存 ring 与独立 render server 的软件虚拟化；
- 可选 process isolation，但不是 hardware passthrough。

与 VirGL 相比，它不试图重建 OpenGL 状态机，而是尽量表达 Vulkan 的显式
对象和同步语义。与 VFIO 相比，Guest 仍然看不到物理 Vulkan device 的 PCI
资源，也无法绕过 Host renderer。

## 7. 控制逻辑

### 7.1 Feature/extension 协商

Host Venus renderer 向 Guest 报告：

- Vulkan XML/API 版本；
- Venus wire/protocol 版本；
- 支持的 extension、feature、limits；
- renderer instance/device 能力。

Guest 看到的 `apiVersion`、`driverVersion` 和 device name 是协议表达，不等
于 Host 物理 GPU 的原始驱动直接透传。

### 7.2 Context 与 ring

Guest 通过 context init 声明 Venus capset、ring 数和 poll mask。Host 创建
proxy context，分配共享 ring/control buffer，建立 Guest/Host 的 ring
sequence 和 fence 状态。

### 7.3 Resource 生命周期

Venus 常见 resource：

- ring/control blob；
- device memory blob；
- mappable shared blob；
- import/export dma-buf；
- image memory、buffer memory。

Host 根据 blob type、flags、blob_id 和 iov 判断 storage 归属。Guest alloc/
free、Host object create/destroy 必须通过 sequence/fence 协调。

### 7.4 Queue submit 与 completion

Guest 更新 ring tail 后，通过一个小 `SUBMIT_3D` 控制包触发 Host 消费。
Host ring 消费位置前移后，Guest 才能回收对应 ring command 和 temporary
shmem。

Vulkan fencing还有独立层级：

```text
ring submit seqno
  -> Host consumed command stream
  -> vkQueueSubmit executed
  -> Vulkan fence/semaphore/timeline signaled
  -> present fence if WSI
```

这三者不能混为一个状态。

## 8. 数据如何传输

### 8.1 Ring data

Venus 把 Vulkan command stream 写入共享 ring。`SUBMIT_3D` payload 可以很小，
因为它主要传递 ring/index/seqno 控制信息。ring 的实际大小和位置由 capset
和 blob mapping 决定。

### 8.2 Shared memory 与 blob

blob 可能表示：

- Guest 分配并附加的 backing；
- Host3D 分配的 image/buffer storage；
- Host-visible mappable region；
- shareable/cross-device handle。

“有 blob create”不等于每块 memory 都经过 `RESOURCE_MAP_BLOB`；descriptor
和 storage 可能通过 fd、opaque handle 或 renderer 内部 object ID 传递。

### 8.3 DMA-BUF/WSI

呈现图像可能通过 dma-buf 交给 Guest compositor。此时必须同时匹配：

- Vulkan format；
- tiling/modifier；
- plane count/offset/stride；
- acquire/release fence；
- compositor 的 DRM format modifier 支持；
- Guest KMS/QEMU scanout 或 cross-domain 路径。

## 9. 同步模型

Venus 涉及四类状态：

| 状态 | 谁维护 | 用途 |
| --- | --- | --- |
| ring head/tail/status | Guest/Host 共享原子变量 | 判断 command 是否被消费 |
| Guest/Host object ref | Venus client/`vkr` | 保证 object 在使用中有效 |
| Vulkan semaphore/fence/timeline | Vulkan objects + Venus sync | queue 间和执行完成 |
| WSI acquire/release/present | WSI + compositor/display | 图像所有权转移 |

Guest Mesa 使用 sync provider 把 Host completion 转回 Guest sync fd/
syncobj。同步错误通常表现为“submit 已出现但等待永不返回”，不能只检查
Vulkan API 返回值。

## 10. 性能模型

Venus 的主要成本点：

- Guest Vulkan command encoding；
- ring 写放大/锁竞争；
- `SUBMIT_3D`/virtqueue notify；
- proxy socket 与 render-server 进程；
- Guest/Host descriptor、object lookup；
- Host Vulkan API validation 和 driver compilation；
- blob import/export、dma-buf 和 WSI；
- ring seqno/fence 等待。

大 command buffer、长 pipeline、compute dispatch 通常比频繁小 submit 更能
摊薄协议成本；但 shader/pipeline 编译和首次 memory allocation 可能成为
长尾。

## 11. 安全与隔离

需要重点验证：

- Guest 构造的 ring descriptor、object ID、blob offset；
- vkr context 资源表是否跨 Guest/context 泄漏；
- render-server 的文件描述符、socket 和 Host Vulkan loader 权限；
- malicious shader/pipeline 对 Host compiler/driver 的影响；
- ring 饱和与内存配额导致的 DoS；
- WSI dma-buf 的 lifetime 和 ownership。

独立 render worker 可以缩小 QEMU 主进程的攻击面，但 Host Vulkan 和物理 GPU
仍是共享资源；这不是 VFIO 的硬件隔离。

## 12. 优点、缺点与适用场景

优点：

- 保留 Vulkan 显式 command/queue/semaphore 语义；
- 适合 Linux Guest/Vulkan 应用；
- 可与 Host Vulkan 多厂商后端共享 GPU；
- 共享 ring 减少控制包大小；
- 可把 Host Vulkan 放入 render server。

缺点：

- Vulkan 对象、descriptor、memory、sync 状态复杂；
- Guest/Host extension、version、device feature 必须匹配；
- WSI 与 dma-buf/modifier 问题是独立故障面；
- ring/seqno/fence 调试成本高；
- render server 和协议是多层信任边界。

适合桌面/开发/云图形和需要 Vulkan 的共享 GPU 场景；不适合要求 PCI 级隔离
或物理 vendor kernel ABI 完整一致性的场景。

## 13. 关键源码入口

- Guest Venus context init：
  `mesa/src/virtio/vulkan/vn_renderer_virtgpu.c`
- Guest ring 创建/提交：
  `mesa/src/virtio/vulkan/vn_instance.c`
  `mesa/src/virtio/vulkan/vn_ring.c`
- Guest device memory：
  `mesa/src/virtio/vulkan/vn_device_memory.c`
- Host object dispatch：
  `virglrenderer/src/venus/vkr_context.c`
- Host ring：
  `virglrenderer/src/venus/vkr_ring.c`
- Host Vulkan device/queue：
  `virglrenderer/src/venus/vkr_device.c`
  `virglrenderer/src/venus/vkr_queue.c`
- Proxy 边界：
  `virglrenderer/src/proxy/proxy_context.c`
- QEMU adapter：
  `qemu/hw/display/virtio-gpu-virgl.c`

## 14. 宏观检查题

1. Guest Vulkan loader 和 Host Vulkan loader 分别在哪里运行？
2. `SUBMIT_3D` payload 为什么可能只有几十字节？
3. ring head 前移、Vulkan fence signal 和 present completion 有什么区别？
4. Venus 如何把 Guest object ID 映射成 Host Vulkan object？
5. 为什么 Venus 同时需要 control virtqueue 和共享 ring？
6. Host3D blob、mappable blob 和 dma-buf 分别解决什么？
7. Guest render server 独立进程能提供什么隔离，不能提供什么？
8. Guest/Host Vulkan extension 不一致时会在哪一层失败？
9. 为什么 swapchain 创建成功不能证明 dma-buf present 路径正确？
10. 如果 `vulkaninfo` 成功但 `vkQueueSubmit` fence 超时，你会检查哪四层？
