# GPU 虚拟化的隔离、安全与故障域

## 1. 文档定位

本文从威胁模型出发，而不是从“某个机制是否名叫 isolation”出发，分析：

- virtio-gpu 软件 mediation；
- native DRM/ROCm context；
- VFIO 整设备直通；
- SR-IOV VF；
- mdev 与 Mediated Pass-Through；
- NVIDIA MIG；
- vendor vGPU；
- MPS/time slicing 等多进程共享方式。

必须同时区分七类隔离：

1. **内存隔离**：一个租户不能读写另一个租户的系统内存或显存。
2. **执行隔离**：work、queue、context 和 engine 状态不能互相污染。
3. **控制面隔离**：Guest 不能控制 Host manager、PF、renderer 或其他 VM。
4. **故障隔离**：一个 context/queue 的错误不能无界扩散。
5. **数据残留隔离**：复用 VRAM、LDS、register、cache 时不泄漏旧数据。
6. **时间隔离**：调度、cache 和带宽竞争受控，侧信道风险可接受。
7. **管理面隔离**：创建、配置、迁移、dump 和 telemetry 不越权。

“安全”只有在明确资产、攻击者、信任边界和可接受风险后才有意义。

## 2. 证据边界与基本原则

本文使用以下公开证据：

- [VFIO 文档](../../linux/Documentation/driver-api/vfio.rst:1)；
- [VFIO mdev 文档](../../linux/Documentation/driver-api/vfio-mediated-device.rst:1)；
- AMDGPU [process isolation](../../linux/Documentation/gpu/amdgpu/process-isolation.rst:1)；
- QEMU virtio-gpu command、resource、blob 和 fence 源码；
- virglrenderer、Rutabaga/Mesa 的协议与对象表源码；
- Linux DMA/IOMMU、PCI AER/reset、KFD、HMM/SVM 文档；
- 已有 GPU 虚拟化平台比较和源码跟踪。

文中将区分：

- **源码证明的机制**；
- **由架构必然存在的风险**；
- **需要具体硬件/产品验证的属性**；
- **不能从名称推出的结论**。

本文不运行恶意 workload、故障注入、侧信道或安全测试。

## 3. 资产与攻击者

### 3.1 需要保护的资产

| 资产 | 泄漏后果 | 破坏后果 |
| --- | --- | --- |
| Host kernel/renderer 内存 | 跨 VM 逃逸 | Host crash、数据破坏 |
| 其他 VM 的 RAM/VRAM | 数据泄漏 | 写入污染、拒绝服务 |
| 模型权重 | 知识产权泄漏 | 模型被复制或篡改 |
| 训练数据 | 隐私/合规泄漏 | 数据投毒、diff 泄漏 |
| Prompt、KV cache、embedding | 用户数据泄漏 | 推理结果污染 |
| credential/token | 租户身份被冒用 | 横向访问 |
| checkpoint | 完整模型/训练状态泄漏 | 恢复错误/恶意注入 |
| queue/fence/event | 调度干扰 | hang、错误完成、死锁 |
| telemetry/trace/dump | 侧信道或敏感数据 | 运营信誉和合规风险 |
| firmware/PF/vGPU manager | 全设备失陷 | 所有租户受影响 |

### 3.2 攻击者模型

#### 恶意 Guest userspace

可能提交：

- 非法 command length/offset/opcode；
- 伪造 resource/context/queue handle；
- 极端尺寸、对齐或整数溢出输入；
- 大量 blob/pin/mapping 请求；
- 恶意 shader/pipeline/descriptor；
- 反复 map/unmap/destroy 制造 UAF 竞态；
- 异常的 fence/timeline dependency。

#### 恶意 Guest kernel

在 VFIO/SR-IOV/vGPU 中拥有较强能力，可能：

- 直接写设备 MMIO；
- 构造任意 queue/doorbell；
- 滥用 ATS/PASID；
- 触发 firmware bug；
- 在 reset 后继续发起 DMA；
- 通过 P2P/ACS 缺口绕过 Host 预期边界。

#### 恶意或脆弱 Host component

包括：

- VMM/QEMU 的设备模型；
- renderer、proxy、shader compiler；
- C/Rust FFI；
- PF/vGPU manager；
- telemetry/agent；
- Host GPU driver/firmware。

即使 Guest 不可信，Host 组件本身也可能有内存安全或逻辑漏洞。

#### 恶意同租户/同 VM workload

同一 VM 内不同进程也可能通过 GPU cache、LDS、register、共享 tensor、
allocator 错误或 debug interface 相互影响。VM 边界不保护 VM 内部的进程。

## 4. 信任边界

### 4.1 Virtio-GPU API remoting

```text
Guest app / malicious userspace
  |
  | normal API + hostile ioctl/protocol inputs
  v
Guest UMD/KMD
  |
  | virtqueue + resource IDs + iovecs + command bytes
  v
QEMU device model
  |
  | validated C ABI / component protocol
  v
renderer/proxy/FFI
  |
  | Host API, dma-buf, driver
  v
physical GPU
```

主要信任边界：

1. Guest 无法信任；
2. QEMU 必须把 Guest 输入当作不可信协议数据；
3. renderer 必须把 QEMU 输入继续当作不可信；
4. Host driver/IOMMU 不应假设 renderer 一定正确；
5. Host GPU firmware 仍需独立安全评估。

将 renderer 放进独立进程可建立进程级边界，但只有配合最小权限、seccomp、
文件系统隔离、IOMMU 和资源限制才有实际安全价值。

### 4.2 VFIO/SR-IOV/vGPU

```text
Guest kernel
  |
  | MMIO, BAR, queue, doorbell, MSI-X
  v
virtual/physical PCI function
  |
  | DMA + interrupt remapping
  v
IOMMU/PF/firmware/GPU engines
```

主要信任边界：

- Guest kernel 与 VMM/Hypervisor；
- VF 与 PF；
- device DMA 与 Host memory；
- VF 与 peer VF/VM；
- GPU firmware 与 platform firmware；
- vGPU manager 与普通 VM。

IOMMU 是 DMA 边界，不是 command processor 安全边界，也不是时间隔离边界。

## 5. 多租户隔离矩阵

| 机制 | 内存/DMA | GPU context | engine 时间 | firmware/reset | 数据残留 | 主要缺口 |
| --- | --- | --- | --- | --- | --- | --- |
| API remoting | Host renderer + IOMMU | Host 对象表 | Host GPU 调度 | Host 设备共享 | Host allocator/cache | parser、兼容性、侧信道 |
| native DRM context | resource/GPU VM | Host native context | Host driver | Host 设备共享 | Host native 资源 | UAPI/驱动耦合、资源隔离 |
| VFIO | Guest GPU VM + IOMMU | Guest 创建物理 context | 整卡共享/独占 | 物理设备 | 整卡内由 Guest driver 管理 | IOMMU group、ACS、reset |
| SR-IOV VF | VF VRAM + IOMMU | VF 硬件 context | PF/GPU 调度 | shared PF/firmware | VF reset/partition | 共享 engine/fabric、配置 |
| mdev | Host 映射/IOMMU | vendor mediated | vendor scheduler | vendor/PF | vendor 实现 | 边界取决于 vendor |
| MPT | direct windows + IOMMU | Guest + Host 控制 | hardware/Host | 依设计 | partition/context | trap 面与共享状态 |
| MIG | HBM/L2 slice | GPU partition context | hardware partition | GPU-level shared | 需 cleaner/partition 保障 | PCIe、管理、time side channel |
| vendor vGPU | Guest VRAM quota | guest+Host manager | Host/vendor scheduler | 依产品 | VRAM/context cleanup | 具体产品依赖 |
| MPS/time slicing | 共享整卡 | 多进程 context | 时间片 | 共享 device | 需 shutdown cleanup | 故障/数据/时间隔离弱 |

没有任何一格可以仅凭功能名填写为“强隔离”。

## 6. Virtio-GPU 控制面攻击面

### 6.1 命令长度和对齐

QEMU 必须验证：

- virtqueue descriptor 是否足够；
- command header 是否完整；
- payload size 是否小于协议上限；
- `iov_to_buf()` 实际复制长度是否等于声明大小；
- command 是否 4-byte/自然对齐；
- 由 Guest 提供的 pointer/offset 是否溢出。

例如：

- [`virtio_gpu_create_mapping_iov()`](../../qemu/hw/display/virtio-gpu.c:983)
  限制 mem entry 数量并用 `g_try_malloc`/`dma_memory_map()`；
- [`rutabaga_cmd_submit_3d()`](../../qemu/hw/display/virtio-gpu-rutabaga.c:348)
  检查 `cs.size`、分配失败和 iov 复制长度；
- [`virgl_renderer_submit_cmd()`](../../virglrenderer/src/virglrenderer.c:319)
  再检查 context 是否存在、长度溢出和 buffer 对齐。

一次检查不够。每穿过一层 ABI，都应验证该层依赖的不变量，因为不同组件
可能被独立升级，且不同 parser 的整数类型/格式不同。

### 6.2 Resource/context 身份

攻击点：

- resource ID 0 或重复创建；
- 未 attach backing 就 transfer；
- transfer 超出 blob size；
- context/resource 跨租户别名；
- destroy 后继续 submit；
- attach/detach 与 pending command 竞态；
- map window offset/size 越界。

源码中有多类边界检查，例如：

- resource 必须存在且类型正确；
- blob backing 不得小于 blob size；
- hostmem map offset + size 不得越过窗口；
- context 不存在时 renderer 返回错误。

安全审查要把“检查存在”和“检查所有权”分开。一个全局 resource table 中
“ID 存在”不代表当前 VM/context 有权访问它。

### 6.3 Capset 和协议版本

Guest 可根据 Host 提供的 capset/protocol version 选择编码方式。版本不一致可能
造成：

- 字段长度解释不同；
- 枚举值含义变化；
- 新 flag 被旧层静默忽略；
- 不同组件使用不同 capability bit。

兼容性检查不是单纯用户体验问题。错误协商可能把合法 Guest 数据解释成
危险 Host 操作。

### 6.4 Fence 和 response

需要验证：

- `fence_id`、`ctx_id`、`ring_idx` 的组合；
- command 与 fence queue 的匹配；
- timeout/error 时是否仍写成功 response；
- Host callback 是否可重入或跨线程；
- response 写入失败后的资源回收；
- 同一 timeline 的乱序/重复 completion。

完成路径既是正确性问题，也是资源释放安全问题。错误 fence 可导致对象过早
释放或 Host 引用长期泄漏。

## 7. Host Renderer 与进程边界

### 7.1 Parser 与 shader compiler

API remoting 把大量不可信输入送入 Host：

- binary command stream；
- pipeline/descriptor metadata；
- shader/kernel bytecode；
- image format/modifier；
- object handle arrays。

边界组件可能需要：

- 内存安全 parser；
- 所有长度/索引/偏移检查；
- shader compiler sandbox；
- 禁止 Guest 控制 path、fd、socket、module；
- 限制 command 递归、对象数量和临时内存；
- 独立 UID/namespace/seccomp；
- crash 后只影响该 renderer/VM。

“virglrenderer 在 QEMU 进程外”只是可能的部署方式，源码本身不自动提供
容器或 seccomp 隔离。

### 7.2 C/Rust FFI

Rutabaga FFI 使用 opaque pointer、显式 length、`catch_unwind()` 和错误码，
降低 Rust panic 穿过 C ABI 的风险。但：

- raw pointer 仍必须由 QEMU 保证有效；
- length 必须来自可信的协议解析；
- `catch_unwind` 不能修复 UB；
- Rust object 被释放后，C 侧不能继续调用；
- C 回调进入 Rust、Rust 回调进入 QEMU 都必须有明确线程/owner 约束。

安全边界是 C ABI 两侧的 contract，不是“用了 Rust”这一事实。

### 7.3 Host resource ownership

Host renderer 通常维护：

- context table；
- resource table；
- dma-buf/opaque fd；
- Host Vulkan/GL object；
- mapping window；
- fence/timeline。

任一 table 的 resource ownership 错误都会产生：

- cross-context access；
- UAF；
- fd 泄漏；
- Host memory disclosure；
- GPU 写错 backing。

对象 ID 应由名称空间/租户/context 绑定，而不只依赖不可猜测 ID。

## 8. 内存与 DMA 威胁

### 8.1 IOMMU 的作用

[VFIO 文档](../../linux/Documentation/driver-api/vfio.rst:5) 明确把 DMA 描述为
设备直通最关键的完整性问题：设备能读写任意 Host memory 时，系统边界会
被破坏。IOMMU 将允许的 IOVA 限制到指定物理页。

但：

- IOMMU 只约束已配置的 DMA 路径；
- 设备内部 VRAM 访问不经过 Host IOMMU；
- PCIe P2P 可能绕过 Host 预期的软件拦截；
- ATS/PASID 配置错误可能改变翻译边界；
- GPU page table 漏洞可能让 GPU 生成未授权 DMA；
- reset 后残留 DMA/TLB 可能继续访问旧页；
- no-IOMMU 模式故意放弃了核心 DMA 隔离。

因此“VFIO 已绑定”只证明 Host driver ownership 和 IOMMU group 关系，不证明
设备全部固件路径安全。

### 8.2 GPU page table 不是 IOMMU 的替代品

Guest 或 renderer 建立的 GPU PTE 决定 GPU VA 指向哪里。Host IOMMU 再决定
GPU 发出的 DMA 能访问哪些 Host 页。

两层都必须正确：

```text
GPU VA --GPU PTE--> device DMA address --IOMMU--> Host page
```

只限制 GPU PTE 而不配置 IOMMU，设备/firmware bug 仍可能访问其他地址。
只配置 IOMMU 而 GPU PTE 管理错误，合法 DMA 仍可能写错 Guest 资源。

### 8.3 映射、pin 与 DoS

恶意 Guest 可请求：

- 极大 blob/VRAM；
- 巨量 small mappings；
- long-term userptr pin；
- 大量 page-table pages；
- 大量 fence/event；
- 大量 queue/context；
- 反复 migration/eviction。

需要配额：

- per-VM/per-tenant；
- per-context/queue；
- HBM/GTT/pinned memory；
- page tables、workspace 和 migration staging；
- fault rate 和 interrupt rate；
- map/unmap/submit rate。

不限制资源数量会把内存安全边界转化为 DoS。

### 8.4 DMA coherence 与 bounce buffer

DMA API 的 bounce buffer/SWIOTLB 会导致数据经过 Host 临时内存。对普通功能，
这只意味着额外拷贝；对机密计算或强数据隔离，它可能意味着 Host 可见明文。

因此：

- `dma_map_*` 成功不等于 Host 不接触数据；
- `DMA_ATTR_REQUIRE_COHERENT` 若导致映射失败，说明平台无法满足 uAPI；
- 忽略失败并退回不具连续 coherence 的映射会破坏语义；
- confidential VM 需要单独证明 DMA bounce/decrypt 路径。

完整地址与 coherence 语义见
[GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)。

## 9. 生命周期、竞态与 UAF

### 9.1 危险状态转换

一个 GPU resource 通常依次经历：

```text
create
  -> attach/map
  -> referenced by queued work
  -> GPU still accessing
  -> detach/unmap
  -> final fence
  -> free
```

危险来自同一对象同时处于多个阶段：

- command 已提交，但 resource 尚未 attach；
- resource unref 后 renderer 仍有 pending work；
- GPU work 完成后，event 还没从 waitqueue 删除；
- Host map 已删除，但 VMM MemoryRegion 仍可能被访问；
- QEMU command suspended，Guest 已销毁 resource；
- Host callback 回到已 reset 的 device。

### 9.2 Unmap 顺序

通用安全方向：

1. 阻止新的 submit/reference；
2. 等待或取消使用该对象的 work；
3. invalidate GPU PTE/TLB；
4. 删除 VMM mapping；
5. DMA unmap；
6. 解除 Host/QEMU object 引用；
7. 最后释放 backing/page。

具体顺序由设备架构决定，但“先释放内存、后停止设备访问”总是重大风险。

### 9.3 Rust/FFI 和异步 callback

Rutabaga 的 fence callback 可能从非 QEMU 线程到达，QEMU 使用 AIO bottom half
把完成转回设备上下文。这在性能上增加一次调度，但避免从任意 backend 线程
直接操作 QEMU RCU、设备状态和 response。

安全审查应检查：

- callback 时 `ctx_id/resource` 是否仍有效；
- callback 是否携带足够 generation/epoch 区分旧设备状态；
- device reset 后旧 callback 是否丢弃；
- component unload 时是否 drain callback；
- C object 释放后是否仍有 Rust/backend 引用。

### 9.4 进程退出

Guest/Host process 退出必须处理：

- queue 停止；
- pending GPU fault 被 drain；
- event waitqueue 唤醒；
- dma-buf/opaque fd 关闭；
- pinned pages 和 DMA mapping 释放；
- PASID/GPU VM unbind；
- renderer context/resource 释放；
- deferred work 不再访问已销毁 process。

用户态在 close fd 后立即假设资源消失，是常见的 UAF 误解。

## 10. 故障域与 reset 隔离

### 10.1 故障域必须写进产品契约

每个方案至少定义：

| 事件 | 预期影响范围 | 恢复动作 |
| --- | --- | --- |
| 非法 command | 当前 submit/context | 返回错误、拒绝命令 |
| GPU page fault | 当前 process/context | fault、迁移或终止 |
| queue hang | queue/engine | preempt、queue reset |
| engine fault | engine/受影响 context | engine reset |
| ECC/poison | allocation/device | 隔离页、reset 或 device lost |
| PCIe AER | link/function/hierarchy | 驱动 `error_detected`/reset |
| PF/firmware fault | VF 或整 GPU | VF stop、PF reset 或整卡恢复 |

### 10.2 PCIe reset

[`pcieaer-howto.rst`](../../linux/Documentation/PCI/pcieaer-howto.rst:123)
说明 PCIe error recovery 需要驱动 callback；fatal error 的 reset 范围可能
涉及 endpoint、upstream port 或更大 hierarchy。

对多 VM GPU，这意味着：

- 一个 VF 的设备错误不一定只影响该 VF；
- FLR 是否足够取决于设备和错误类型；
- VMM 必须处理 host-side AER/reset notification；
- Guest 内 reset 和 Host 物理 reset 不能简单等同。

### 10.3 Reset 后残留

reset 流程应确认：

- bus mastering/DMA 已停止；
- outstanding TLB 已失效；
- queue/doorbell 不再接受新 work；
- completion/IRQ 不再指向旧 event；
- VRAM/context save/scratch 不会映射给下一个租户；
- Host PF 已重建必要 VF 状态。

没有 reset completion 证据时，不能把设备重新分配给新租户。

## 11. 数据残留和 GPU state

### 11.1 为什么 VRAM 清零不够

旧 workload 的数据可能位于：

- VRAM/HBM allocation；
- shader LDS/shared memory；
- GPR/vector register；
- cache/L2；
- command buffer；
- context save area；
- page tables；
- staging/bounce buffer；
- crash dump/trace buffer；
- allocator free list metadata。

只清 VRAM 不等于清理所有微架构状态。

### 11.2 AMDGPU process isolation

[AMDGPU process isolation](../../linux/Documentation/gpu/amdgpu/process-isolation.rst:8)
明确说明其机制会序列化 graphics engine 访问，并在 jobs 之间运行 cleaner
shader 清除 LDS 和 GPR；支持 partition 的设备可按 partition 启用。

这提供三个重要结论：

1. pass-through/VF 也未必自动在 process 间清 GPU 内部状态；
2. scheduler 和 context switch 不等于数据残留清理；
3. 数据残留策略可能需要性能/隔离 trade-off。

是否覆盖 compute、其他 ASIC、cache 和所有 engine，要按具体配置和文档确认。

### 11.3 释放后的显存

VM 终止或资源释放时，Host allocator 应：

- 等 pending DMA 完成；
- 清 PTE/TLB；
- 按威胁模型清零、加密或确保旧页不可恢复；
- 清除 dma-buf import/export 引用；
- 确保崩溃/kdump 不包含租户敏感数据。

“物理页即将分配给另一个租户”是数据残留边界，而不仅是 allocator 性能问题。

## 12. 时间与微架构侧信道

### 12.1 共享资源

GPU 多租户可能共享：

- shader array；
- L1/L2；
- HBM controller/bandwidth；
- copy engine；
- command processor；
- scheduler/firmware；
- PCIe/fabric；
- power/thermal budget。

共享会增加：

- cache contention channel；
- timing channel；
- bandwidth/power covert channel；
- priority/observability inference。

### 12.2 隔离强度不是二值

即使 MIG 在硬件上切分 SM/L2/HBM，仍可能存在共享 PCIe、firmware、管理接口
或平台侧通道。VF 也可能共享 engine 调度、HBM 带宽和 reset 域。

安全声明应写清：

- 防的是 accidental interference 还是 malicious side channel；
- 是否有噪声/bandwidth 上限；
- 是否经过具体攻击模型验证；
- 是否接受残余风险。

### 12.3 优先级和调度也是安全参数

恶意租户可尝试：

- 高优先级持续占用；
- 大量 queue 占满 slot；
- 高频 fence/interrupt 占用 Host CPU；
- 长 kernel 阻止 preemption；
- 反复 migration 造成抖动。

资源隔离必须包括 scheduler、interrupt、fault handling 和 management thread。

## 13. 各虚拟化机制的安全边界

### 13.1 Virtio-GPU API remoting

主要边界：

- Guest 无法直接写物理 GPU MMIO；
- Host renderer 解析 API command；
- resource/blob/dma-buf 由软件映射；
- Guest/Host object table 分离。

主要风险：

- Host parser/compiler 漏洞；
- resource 身份混淆；
- UAF/mapping race；
- renderer 权限过大；
- Host GPU 仍共享；
- 侧信道与时延；
- API/库兼容性被误认为隔离。

适合把 renderer 部署到受限进程/容器，并为每个 VM 或信任域配置资源上限。

### 13.2 Native DRM/ROCm context

缩小 API 语义差距，但 Host native driver 仍解析更复杂的 UAPI：

- GPU VM/BO map；
- command submission；
- queue/event/syncobj；
- userptr/SVM。

它可能提高兼容性，却不会自动缩小 Host parser/firmware 攻击面。

### 13.3 VFIO

优势：

- Guest 原生 driver；
- IOMMU 限制 DMA；
- Host 不需要解析 GPU command；
- 通常一整卡给一个 VM，租户间共享路径少。

边界条件：

- IOMMU group 可由 multifunction/bridge non-ACS backdoor 放大；
- 配置空间、ROM、MSI-X、FLR 和 AER 必须正确处理；
- GPU firmware、command processor 和 display 仍需安全评估；
- Host 可能失去设备，管理面复杂度高；
- live migration 通常不具备或不成熟。

### 13.4 SR-IOV VF

优势：

- 物理 PCIe function 边界；
- Guest 原生 driver；
- 通常配合 VFIO/IOMMU assignment；
- PF 管理 VF 生命周期和资源配额。

仍需验证：

- VF 间 peer/P2P/ACS；
- shared firmware/scheduler；
- VF reset/error containment；
- HBM quota 和 residual data；
- PF manager 权限与迁移。

VF 是强资源边界，不是“完整安全认证”的同义词。

### 13.5 mdev

[VFIO mdev 文档](../../linux/Documentation/driver-api/vfio-mediated-device.rst:1)
提供统一的 mediated device 管理/IOMMU group 集成，但每个 vendor driver 决定：

- MMIO trap/模拟范围；
- queue/doorbell 代理；
- memory window 和 dma-buf；
- reset/error；
- resource quota；
- migration。

因此 mdev 的安全属性来自 vendor implementation，而不只来自 mdev 框架。

### 13.6 MPT

MPT 的威胁模型取决于直通面：

- 哪些 BAR 可写；
- 哪些 queue/doorbell 直接映射；
- GPU VM/page table 归谁管理；
- 控制寄存器是否 trap；
- Host 如何做 reset/error recovery。

直通越多，性能越好，但 Guest kernel/firmware 可触发的硬件状态越多。

### 13.7 MIG

优势：

- 硬件划分 SM/L2/HBM；
- 可减少部分资源竞争；
- profile 可约束计算和显存能力。

限制：

- 不是 PCIe VF；
- 不提供完整 VM 管理面；
- reset/firmware/PCIe 可能仍共享；
- P2P/media/engine 支持受 profile 限制；
- 时间侧信道和共享控制路径仍需评估；
- 不能仅凭 MIG 声称完整 confidential isolation。

### 13.8 Vendor vGPU

vGPU 可能提供：

- Guest vendor driver；
- 虚拟 aperture；
- Host scheduler；
- VRAM quota；
- 管理/管理 API；
- reset/迁移能力（依产品）。

安全验证必须绑定：

- GPU 产品与 mode；
- vGPU manager 版本；
- Guest/Host driver；
- Hypervisor；
- 是否 SR-IOV/mediated；
- 是否支持 confidential computing；
- 已知漏洞和 certification 状态。

### 13.9 MPS/time slicing

MPS 或 time slicing 可以减少 launch overhead 或提高利用率，但通常共享：

- GPU context/scheduling subsystem；
- 同一 device memory system；
- engine 和 scheduler；
- 故障域；
- 数据残留清理要求。

高安全多租 VM 不应仅依赖 MPS/time slicing。

## 14. AI 数据安全

### 14.1 数据流

AI 推理的敏感数据可能经过：

```text
network
  -> tokenizer / preprocessing
  -> Guest CPU memory
  -> pinned staging
  -> H2D copy / mapped memory
  -> model input / KV cache
  -> GPU compute
  -> logits / output
  -> D2H copy
  -> telemetry / response
```

每个跨边界步骤都可能有日志、dump、trace 或 bounce buffer。

### 14.2 模型权重

保护要求：

- 只允许目标 VM/tenant 访问；
- Host allocator 释放后不可被下一租户读；
- checkpoint/export 需认证和加密；
- trace 不记录大块 tensor；
- 侧信道不能恢复权重；
- model cache 不跨租户错误复用。

### 14.3 Prompt、embedding 和 KV cache

KV cache 有以下安全特点：

- 生命周期与请求/会话相关；
- 可能包含完整上下文；
- 常驻 HBM，不允许随意 evict；
- 动态增长，allocator 容易复用错误；
- 可被 debug/profiler 接口意外暴露。

资源复用必须显式清理或重新初始化，而不能依赖“新 allocation 应该有正确
形状”。

### 14.4 训练数据和梯度

训练环境还要保护：

- 样本 tensor；
- activation；
- gradient；
- optimizer state；
- differential privacy 相关 state；
- RNG 状态；
- collective payload；
- checkpoint/RNG/optimizer consistency。

一个集体通信配置错误可能把 gradient 发送给错误 rank 或 Host。

### 14.5 telemetry 和 crash 证据

安全与可观测性有冲突：

- trace 需要 resource/queue/tensor 信息；
- crash dump 需要 GPU memory/context；
- profiler 需要 shader/kernel 信息；
- 但这些数据可能包含模型和用户输入。

需要定义：

- 谁能读取 telemetry；
- retention；
- 脱敏/加密；
- 租户是否 opt-in；
- 跨租户全局 admin 视图的权限。

## 15. Confidential computing 的边界

CPU TEE、GPU private HBM、DMA/SWIOTLB、attestation 和 AI 数据生命周期的深入
分析见
[Confidential GPU 与机密 AI 虚拟化](confidential-gpu-virtualization.md)。

### 15.1 CPU confidential VM 不等于 GPU confidential

如果 VM 内存受加密和 attestation 保护，但 GPU workload 还需要 Host
renderer/API remoting：

- Host renderer 可能解析 kernel/command；
- Host driver 可能看见 buffer 或 relocation；
- bounce buffer 可能是明文；
- Host completion/fence 仍由 Host 管理；
- GPU MMIO/BAR 仍可能由 Host 访问。

因此“Guest CPU memory encrypted”不能推出“GPU 模型/HBM confidential”。

### 15.2 GPU confidential computing 需要端到端属性

至少验证：

- GPU private memory 是否加密；
- Host 是否可读 VRAM/context save；
- DMA shared/private page 区分；
- command/compute path 是否保持机密；
- attestation 覆盖 GPU/firmware/config；
- reset/迁移/管理面；
- telemetry/crash；
- peer GPU/collective 是否破坏边界。

具体产品能力和限制必须使用厂商官方安全模型，而不能从通用 CPU CC 文档外推。

## 16. 管理面安全

### 16.1 管理接口

包括：

- VFIO group/device 节点权限；
- mdev create/remove sysfs；
- PF/VF 配置；
- vGPU manager API；
- GPU reset 和 power controls；
- telemetry/debug/profiler；
- device plugin/DRA/Kubernetes 权限；
- 镜像和 renderer 升级。

能够创建或重置 GPU 设备的主体，通常已经拥有比普通 workload 更高的权限。

### 16.2 热插拔与重分配

流程应确认：

- VM 已停止访问旧 device；
- queue/DMA/IRQ/TLB 已 drain；
- context/VRAM 已清理；
- Guest-visible ID/generation 不混淆；
- 新 tenant 不会收到旧 fence/event；
- hotplug ABI 与 Guest driver 顺序正确。

### 16.3 Live migration

迁移数据可能包含：

- GPU VM/PTE；
- queue/context save；
- VRAM；
- dma-buf backing；
- fence/timeline；
- display state；
- checkpoint data。

必须保护：

- source/destination 认证；
- channel 加密；
- 目的地状态验证；
- stale/failed migration rollback；
- 旧 Host 的清理；
- 跨 profile/device 兼容性；
- tenant 对迁移的可见性。

## 17. 威胁建模工作表

### 17.1 资产和攻击者

```text
Protected assets:
  tenant data / weights / prompts / KV / checkpoints / credentials

Adversary:
  guest userspace / guest kernel / other tenant / host component / admin

Capabilities:
  MMIO / command bytes / DMA / timing / telemetry / reset

Trust boundaries:
  user/kernel / guest/host / VF/PF / tenant/manager / GPU firmware
```

### 17.2 控制面

```text
Input validation:
  opcode / size / offset / alignment / ID / ownership / version

Parser isolation:
  process / uid / namespace / seccomp / memory limit / restart

Resource ownership:
  tenant namespace / context / generation / ref / teardown
```

### 17.3 内存

```text
GPU PTE owner:
IOMMU domain:
ATS/PASID/P2P policy:
Pin/migration/quota:
DMA coherence/bounce:
Residual data cleanup:
Crash/trace redaction:
```

### 17.4 执行与故障

```text
Queue/context isolation:
Preemption:
Reset scope:
Error notification:
Stale callback/epoch:
Tenant recovery:
```

### 17.5 侧信道

```text
Shared cache/bandwidth:
Priority/fairness:
Microarchitectural cleanup:
Observable counters:
Accepted residual risk:
```

## 18. 安全评审检查表

1. 是否列出 Guest 可控制的每个长度、ID、offset、handle 和 flag？
2. 每层 parser 是否验证自己依赖的不变量，而不是信任上一层？
3. resource ID 是否包含租户/context 所有权，而非仅全局存在性？
4. map/unmap/unref 与 pending command 的时序是否有明确状态机？
5. Host callback 是否有 generation/epoch 防止 reset 后旧事件生效？
6. C/Rust FFI 的 pointer、length、ownership、panic 和线程约束是否完整？
7. renderer 是否运行在最小权限环境中，能否被隔离和重启？
8. IOMMU domain、IOVA 权限和 no-IOMMU 策略是否与威胁模型一致？
9. GPU PTE、IOMMU、ATS/PASID 和 P2P/ACS 是否分别验证？
10. HBM、GTT、pin、page table、fence、queue 和 fault rate 是否有 quota？
11. DMA bounce、trace、dump、checkpoint 是否泄露模型或用户数据？
12. VF/MIG/vGPU 是否明确共享哪些 firmware、engine、PCIe 和 reset 域？
13. process/context 切换是否清理 LDS、GPR、cache、VRAM 和 context save？
14. reset/error 是否定义范围，并有停止 DMA、失效 TLB、重建对象的顺序？
15. 热插拔、迁移和重分配前是否 drain 旧工作并清除旧数据？
16. telemetry/profiler/debug interface 是否等权保护为生产数据通道？
17. confidential computing 声明是否覆盖 GPU/HBM/firmware/DMA/attestation，
    而不是只覆盖 Guest CPU 内存？

## 19. 常见错误结论

### “有 IOMMU 就安全”

错。IOMMU 限制 DMA 翻译，但无法修复 parser、GPU firmware、VRAM 隔离、
reset 残留和侧信道。

### “renderer 在 Rust 中，所以 parser 安全”

错。Rust 能减少一部分内存安全风险，不安全 FFI、逻辑漏洞、资源耗尽和 Host
API 误用仍然存在。

### “有 mdev/VF/MIG 就自动隔离所有租户”

错。必须检查 vendor driver、硬件 profile、共享 firmware/reset、数据清理和
管理面。

### “MPS 或 time slicing 可以提供 VM 安全隔离”

错。它们主要改善共享/利用率，不是完整 VM 威胁模型。

### “VM 退出后 destroy 返回成功，GPU 数据已清理”

错。仍需 drain DMA、invalidate TLB、清理 VRAM/context/scratch，并避免
dump 或 allocator reuse 泄漏。

### “CPU confidential VM 让 GPU workload 也机密”

错。只有 GPU memory、command、DMA、firmware、attestation 和 telemetry
路径都具有相应属性时，才可能成立。

## 20. 与其他专题的关系

- 显存、DMA 和 coherence：  
  [GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)
- 执行、preemption、fault 和 reset：  
  [GPU 执行模型、队列、调度与故障隔离](gpu-execution-model.md)
- 硬件/软件/API 方案分层：  
  [GPU 虚拟化平台比较](../virtio-gpu/platform-comparison.md)
- 四路径 parser/对象/fence 源码：  
  [四路径源码横向对照](../virtio-gpu/source-trace-cross-path.md)
- AI workload 与安全评估：  
  [AI GPU 计算栈与虚拟化边界](ai-compute-virtualization.md)
- 多卡通信与 P2P 边界：  
  [NCCL/RCCL Collective 源码跟踪](nccl-rccl-source-trace.md)

## 21. 检查题

1. 为什么 VM 隔离、进程隔离、GPU context 隔离和 IOMMU 隔离不是同一件事？
2. Guest 能控制哪些 virtio-gpu command 字段，QEMU 和 renderer 分别验证什么？
3. 为什么“resource ID 存在”不能证明当前 context 有权访问该 resource？
4. `catch_unwind()` 能保护什么，不能保护什么？
5. VFIO IOMMU group 为什么可能大于单个设备？
6. PCIe ACS、P2P、ATS/PASID 如何影响设备直通边界？
7. GPU page table 和 Host IOMMU 为什么必须同时正确？
8. long-term userptr pin 如何形成内存 DoS？
9. map/unmap/unref 与 pending GPU work 的安全顺序是什么？
10. 为什么 GPU LDS/GPR 清理需要单独机制，context switch 可能不够？
11. VF、MIG、MPS 和 vGPU 分别共享哪些执行、内存和故障资源？
12. SR-IOV VF 为什么仍可能需要检查 shared firmware 和 reset 域？
13. reset 后为什么要处理 stale IRQ、event、fence、TLB 和 DMA？
14. KV cache 和 checkpoint 的访问、复用与清理有什么不同要求？
15. CPU confidential VM 为什么不能自动保证 GPU HBM 和模型权重机密？
16. telemetry、profiler 和 crash dump 为什么也属于生产安全边界？
