# Venus 源码跟踪

## 1. 阅读范围

对应宏观文档：[venus-overview.md](venus-overview.md)。Venus 额外依赖
resource blob、context ring index 和共享 ring；这些公共 KMD 机制见
[shared-kmd-source-trace.md](shared-kmd-source-trace.md)。

本文跟踪 Vulkan API call 如何变成 Venus wire command、写入共享 ring，
再由 Host `vkr` 调用 Host Vulkan。源码基线同
[四路径源码横向对照](source-trace-cross-path.md)。

宏观问题与本文源码入口的对应关系如下：

| 宏观问题 | Venus 的源码答案 | 本文位置 |
| --- | --- | --- |
| Guest UMD 做什么 | Mesa Venus ICD 建立 renderer/ring，编码 Vulkan protocol command | 第 3–5 节 |
| Guest KMD 做什么 | 建 context/blob/mapping，运输少量控制包、ring notify 和 fence | 第 6 节、共用 KMD |
| 如何利用 Host GPU | proxy 把 ring 送到 render server，`vkr` 解协议并调用 Host Vulkan | 第 7–8 节 |
| 虚拟化形式 | Vulkan command remoting；大命令走共享 ring，控制信息走 virtio | 第 2、5–8 节 |
| 数据如何传输 | head/tail shared ring、optional notify control command、blob/dma-buf | 第 3–5、9 节 |
| 控制逻辑 | capset/context/ring 建立由 Guest 发起，Host dispatch 建真正 Vulkan 对象 | 第 3–8 节 |
| 内存与所有权 | Host Vulkan memory 对象与 Guest blob/BO/dma-buf backing 分离 | 第 9 节 |
| 同步与完成 | ring head/seqno、VkFence、semaphore/timeline、sync fd 分属不同阶段 | 第 10 节 |
| 呈现 | Guest WSI/dma-buf 与 acquire/release fence 决定合成可见性 | 第 11 节 |

## 2. 总调用链

```text
vkCreateInstance
  -> vn_CreateInstance()
  -> vn_instance_init_renderer()
  -> virtgpu_ioctl_get_caps/getparam/context_init
  -> vn_instance_init_ring()
  -> vn_ring_create()

vkCmd*/vkQueueSubmit
  -> Venus protocol encoder
  -> vn_ring_submit_*()
  -> write shared ring + store tail
  -> optional vkNotifyRingMESA control command
  -> virtio_gpu
  -> QEMU
  -> virglrenderer proxy_context
  -> render server
  -> vkr
  -> Host Vulkan
```

## 3. Guest UMD 初始化

### 3.1 Instance

[`vn_CreateInstance()`](../../mesa/src/virtio/vulkan/vn_instance.c:260)
的顺序：

1. 建立 Vulkan dispatch/extension 表。
2. 检查 app API version。
3. `vn_instance_init_renderer()` 连接 virtio-gpu。
4. 初始化 protocol info 和 shmem pools。
5. `vn_instance_init_ring()` 创建 main ring。
6. 校验 Host renderer 的 wire/XML/protocol version。

### 3.2 Renderer 与 capset

[`virtgpu_init_params()`](../../mesa/src/virtio/vulkan/vn_renderer_virtgpu.c:1039)
要求：

- `3D_FEATURES`
- `CAPSET_QUERY_FIX`
- `RESOURCE_BLOB`
- `CONTEXT_INIT`
- Host-visible 或 Guest VRAM 之一

[`virtgpu_init_capset()`](../../mesa/src/virtio/vulkan/vn_renderer_virtgpu.c:1011)
查询 `VIRTGPU_DRM_CAPSET_VENUS`，拒绝 wire version 0。

### 3.3 Context init

[`virtgpu_ioctl_context_init()`](../../mesa/src/virtio/vulkan/vn_renderer_virtgpu.c:139)
设置：

```text
CAPSET_ID = VENUS
NUM_RINGS = 64
POLL_RINGS_MASK = 0
```

`POLL_RINGS_MASK=0` 表示不使用 drm event 通知 fence，Venus 主要依靠 ring
seqno/sync provider。它并不意味着不存在 timeline。

### 3.4 Ring layout

[`vn_ring_get_layout()`](../../mesa/src/virtio/vulkan/vn_ring.c:256) 把共享 blob
组织为 cache-line 对齐的 head/tail/status 和环形 command buffer，尾部还有
extra 区。

Guest `vn_ring_shared` 保存指向共享原子的 head/tail/status 和 buffer。默认
主 ring buffer 是 128 KiB。

## 4. Ring 内存序

### 4.1 Guest 写 tail

[`vn_ring_store_tail()`](../../mesa/src/virtio/vulkan/vn_ring.c:102) 使用
`memory_order_seq_cst`。原因是 Guest driver 与 Host renderer 可能由不同
编译器实现，需要确保 command bytes 的 cache/write 顺序早于 tail 发布。
Host 端
[`vkr_ring_load_tail()`](../../virglrenderer/src/venus/vkr_ring.c:69) 使用
acquire load。

### 4.2 Host 写 head

Host [`vkr_ring_store_head()`](../../virglrenderer/src/venus/vkr_ring.c:60)
使用 release store。Guest
[`vn_ring_load_head()`](../../mesa/src/virtio/vulkan/vn_ring.c:92) 使用 acquire
load，随后才能回收对应 shmem/command。

这意味着：

```text
tail 可见 ≠ Host 执行完成
head 前移   ≈ Host 已消费 command
fence signal = Vulkan/Host GPU 完成
```

## 5. 写 Ring 与通知

### 5.1 Command bytes

Venus encoder 生成 command stream。ring 有直接写和 indirect/upload 两条路径：

- 小/普通 command 直接写入 ring。
- 大 command 或 pointer storage 先写进 upload shmem，再以引用方式提交。

[`vn_ring_write_buffer()`](../../mesa/src/virtio/vulkan/vn_ring.c:130) 处理
环形 buffer 回绕，并检查 head 释放出的空间。

### 5.2 空间不足

[`vn_ring_wait_space()`](../../mesa/src/virtio/vulkan/vn_ring.c:231) 轮询 Host
head。等待策略不是直接陷入 Host call，而是使用 relax 逻辑，避免每次无空间
都增加一次昂贵往返。

### 5.3 提交和唤醒

[`vn_ring_submit_locked()`](../../mesa/src/virtio/vulkan/vn_ring.c:630)：

1. 判断能否 direct submission。
2. 不能 direct 时 upload command。
3. 准备 ring submission 和需要保活的 shmem。
4. `vn_ring_submit_internal()` 写 command、发布 tail。
5. 若 Host ring 处于 idle，编码 `vkNotifyRingMESA` 通过 renderer control
   path 唤醒 Host。
6. 返回 ring seqno。

所以普通 `SUBMIT_3D` 可以是 24 字节左右的控制包，而真正 Vulkan command
stream 位于共享 ring。

## 6. Guest KMD 与 QEMU

Guest KMD 通过 `CONTEXT_INIT`、resource blob、mapping、`SUBMIT_3D` 和 fence
承载 Venus；它不解析 Vulkan protocol。

QEMU 的 VirGL adapter 对 `SUBMIT_3D`：

1. 校验 payload 上限。
2. 将 Guest SG payload 复制到临时 buffer。
3. `virgl_renderer_submit_cmd()`。

virglrenderer 根据 context 的 capset 选择
[`proxy_context_submit_cmd()`](../../virglrenderer/src/proxy/proxy_context.c:270)。
它发送 `RENDER_CONTEXT_OP_SUBMIT_CMD` 到 render server socket；超过 inline
容量的 command 再发一段 large payload。

## 7. Proxy Context

[`proxy_context_create()`](../../virglrenderer/src/proxy/proxy_context.c:682)：

1. 通过 proxy client 创建 Host context fd。
2. 建立 proxy context 和 socket。
3. 分配 timeline shmem。
4. 建立 fencing/eventfd 和 resource table。
5. 向 render server 发 `RENDER_CONTEXT_OP_INIT`。

Proxy timeline shmem 保存每个 ring 的 current seqno。Host completion
线程检查 timeline，把 fence 完成回送 QEMU。

## 8. Host `vkr` 如何执行

### 8.1 Dispatch table

[`vkr_context_init_dispatch()`](../../virglrenderer/src/venus/vkr_context.c:74)
注册所有 protocol handler，包括：

- instance、physical/device；
- queue、fence、semaphore、event；
- device memory、buffer、image；
- descriptor、pipeline、command buffer；
- WSI/host copy；
- acceleration structure。

### 8.2 解 command

[`vkr_ring_submit_cmd()`](../../virglrenderer/src/venus/vkr_ring.c:215) 把 ring
command 设为 decoder stream，在循环中调用 `vn_dispatch_command()`。

每消费一条 command，它会：

1. 检查 decoder fatal state。
2. 调用对应 Host Vulkan dispatch handler。
3. 立即更新共享 head。
4. 通知 `vkr_context` ring seqno 变化。

这里“head 更新”可能发生在 GPU work 完成之前，因为它只表示 Host 已经读取
并调用 command。

### 8.3 Host objects

Guest object ID 进入 `vkr_context` object table；Host handler 创建真正的
VkInstance/VkDevice/VkQueue/VkImage 等对象。Guest handle 永不等同于 Host
pointer/handle。对象销毁必须先从 table 移除并确保 pending command 不再
引用。

## 9. Device memory

### 9.1 Guest 分配

[`vn_AllocateMemory()`](../../mesa/src/virtio/vulkan/vn_device_memory.c:356)
根据 memory type 和 export/import capability 走不同路径：

- 普通 allocation：经 ring 调 Host `vkAllocateMemory`，异步保存 ring seqno；
- export allocation：在 Guest 创建 blob/BO，再把 resource ID 导入 Host；
- Guest VRAM：Host-visible/guest-vram blob；
- dma-buf import：Guest 先 import fd，再让 Host 通过 resource ID import；
- Android AHB：Android 专用 import。

[`vn_device_memory_bo_init()`](../../mesa/src/virtio/vulkan/vn_device_memory.c:76)
在 Host allocation 还没被 ring seqno 保证完成时，先提交
`vkWaitRingSeqnoMESA`，再创建 Guest blob。

### 9.2 Host 分配

Host `vkr_dispatch_vkAllocateMemory()` 读取
`VkImportMemoryResourceInfoMESA`：

- 有 resource ID 时，从 `vkr_resource` 取 dma-buf/opaque fd；
- 构造 `VkImportMemoryFdInfoKHR`；
- 调 Host `vkAllocateMemory()`；
- 没有 resource 时由 Host Vulkan 正常分配。

这展示 Venus 的混合内存模式：Vulkan memory 对象由 Host 拥有，而 backing
可能来自 Guest blob、Host3D 或 dma-buf。

## 10. Fence、semaphore 和 timeline

Venus 需要映射多层对象：

| Guest 对象 | 协议/Host 对象 | 完成含义 |
| --- | --- | --- |
| ring seqno | shared head/timeline | Host 消费 ring |
| VkFence | Host VkFence | queue submission 完成 |
| VkSemaphore | Host semaphore | queue 间依赖 |
| timeline semaphore | Host timeline/value | 指定 value 完成 |
| sync fd | import/export handle | 跨 API/进程同步 |
| present fence | WSI acquire/release | 图像可复用/可显示 |

[`vkr_context_submit_fence()`](../../virglrenderer/src/venus/vkr_context.c:138)
对 ring 0 直接走 CPU timeline；其他 ring 使用 proxy/renderer timeline。

Guest 的 ring head 不能用于等待 GPU completion，只能用于等待 Host 消费速率。

## 11. WSI 和 dma-buf

`vn_device_memory.c` 和 `vkr_device_memory.c` 展示三种 backing：

- `VIRGL_RESOURCE_FD_DMABUF` -> Host `DMA_BUF_BIT_EXT`
- `VIRGL_RESOURCE_FD_OPAQUE` -> Host `OPAQUE_FD_BIT`
- udmabuf/GBM 路径 -> 创建线性 dma-buf 供 Host import

WSI 还需要 format、modifier、plane 和 acquire/release fence。`vkCreateSwapchain`
返回成功只说明对象/资源初始化完成，不说明 Guest compositor 能采样或显示。

## 12. 生命周期与故障定位

| 阶段 | 主机侧对象 | 失败表现 |
| --- | --- | --- |
| renderer connect | virtgpu/vkr instance | ICD 不可用 |
| context init | proxy context | device 创建失败 |
| ring create | Guest/Host ring | submit 卡在空间或 notify |
| queue submit | Host VkQueue | VkFence timeout |
| memory import | Host device memory | allocation/invalid fd |
| present | swapchain + compositor | 黑屏/内容旧 |

建议 trace 点：

- Guest `VIRTGPU_EXECBUFFER` 和 `virtio_gpu_cmd_ctx_submit`；
- guest/host ring tail/head；
- proxy `RENDER_CONTEXT_OP_SUBMIT_CMD`；
- `vkr_ring_submit_cmd()` command dispatch；
- Host fence callback/QEMU fence response；
- WSI acquire/release。

## 13. 推荐源码阅读顺序

1. `vn_instance.c`：instance、renderer、ring 初始化。
2. `vn_renderer_virtgpu.c`：capset、params、context、ring control。
3. `vn_ring.c`：布局、内存序、等待、提交。
4. `vn_device_memory.c`：blob/dma-buf/import/export。
5. QEMU `virtio-gpu-virgl.c`：submit 控制面。
6. `proxy_context.c`：render server 边界。
7. `vkr_context.c` / `vkr_ring.c`：Host dispatch。
8. `vkr_queue.c` / `vkr_device_memory.c`：Host Vulkan 对象。

## 14. 检查题

1. `vn_ring_shared` 的 head/tail/status 分别由哪端写？
2. 为什么 tail 发布必须与 command write 有明确内存序？
3. ring head 前移为什么不是 VkFence signal？
4. `vkNotifyRingMESA` 与 `SUBMIT_3D` 各自是什么控制动作？
5. 大 command 的 indirect/upload 路径为什么需要额外 shmem 保活？
6. `vkAllocateMemory` 为何可能同时涉及 ring command 和 blob resource？
7. dma-buf import 时 Guest 和 Host 各自持有哪类 resource ID/fd？
8. WSI 的 acquire/release fence 如何影响 compositor？
9. render server 独立后，QEMU 与 `vkr` 的共享对象如何跨进程传递？
10. 如何在源码中区分“Host 已消费 command”和“Host GPU 已执行完”？
