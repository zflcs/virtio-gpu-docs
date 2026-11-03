# Virtio-GPU Native DRM Context 宏观文档

## 1. 定位

Native DRM context 是 virtio-gpu 的扩展对照路径，不属于当前四主路径
`VirGL / Venus / GFXStream / Rutabaga`。它解决的是另一类问题：

> 不把某个高层图形 API 作为唯一 remote contract，而是代理某个 Host native
> DRM 驱动附近的 UAPI/命令模型。

它不是 PCI passthrough：

- Guest 没有物理 GPU 的 BAR、PCI config、MSI-X 或 IOMMU group；
- 物理 GPU 仍由 Host native driver 管理；
- Guest 命令仍经过 virtio-gpu、QEMU 和 virglrenderer；
- reset、错误和生命周期仍由虚拟设备/Host renderer 负责。

## 2. 端到端架构

```text
Guest application / Mesa UMD
  -> native vendor-ish userspace shim
  -> vdrm control-command protocol
  -> virtio-gpu DRM capset 6
  -> VIRTGPU_EXECBUFFER
  -> virtio control virtqueue
  -> QEMU virtio-gpu-gl
  -> virglrenderer DRM native context
  -> Host native backend
  -> Host libdrm + vendor KMD
  -> physical GPU
```

典型 AMD 图形实现：

```text
Mesa AMD Vulkan/Graphics stack
  -> amdgpu_virtio shim
  -> vdrm
  -> virtio-gpu
  -> virglrenderer drm/amdgpu
  -> Host libdrm_amdgpu + amdgpu KMD
```

这里的 “native” 指 native driver UAPI/对象模型，而不是“Guest 直接拥有设备”。

## 3. Guest UMD 与 KMD

### 3.1 Guest UMD

Guest 应用仍使用厂商/图形 API。Guest UMD 需要：

- 通过 virtio-gpu render node 发现 capset；
- 选择匹配的 context type；
- 把原 native driver 对象转换为 control commands；
- 维护 Guest handle、resource ID、GPU VA 和 timeline；
- 通过 `VIRTGPU_EXECBUFFER` 提交 CS。

AMD 的 `amdgpu_virtio` shim 实现一套类似 libdrm_amdgpu 的接口，使上层 Mesa
代码不必为每个调用直接理解 virtio。

### 3.2 Guest KMD

Guest KMD 仍是 virtio-gpu DRM driver。它：

- 暴露 `VIRTGPU_DRM_CAPSET_DRM`；
- 支持 `CONTEXT_INIT`、blob、resource 和 execbuffer；
- 不解析 AMDGPU native command 的 CS 语义；
- 通过 fence FD/syncobj 提供 Guest completion。

Guest KMD 不理解“这是不是一款 AMD GPU 的 CS packet”；它只把 renderer
需要的 command bytes、resource references 和 fences 传给 Host。

## 4. Control shmem

Native context 需要一个小型请求/响应通道：

```text
Host-visible mappable blob
  +--------------------------+
  | request buffer           |
  | response buffer          |
  | host processed seqno     |
  +--------------------------+
```

Guest 调用 `vdrm_alloc_rsp()` 在 response 区分配空间，并在 request 中记录
`rsp_off`。Host 必须在写 response 前验证：

- request length；
- response offset/alignment；
- response length；
- 不覆盖其他 request；
- Guest 修改共享内存造成的 TOCTOU。

Host 使用 shadow response buffer，处理完成后再复制到共享区，避免直接信任
Guest 提供的目标区域。

## 5. 控制命令模型

通用 request：

```text
struct vdrm_ccmd_req
  cmd
  len
  seqno
  rsp_off
```

Response 头包含长度。不同 vendor backend 定义自己的 opcode。

AMDGPU backend 当前包括：

- QUERY_INFO；
- GEM_NEW；
- BO_VA_OP；
- CS_SUBMIT；
- SET_METADATA；
- BO_QUERY_INFO；
- CREATE_CTX；
- RESERVE_VMID；
- SET_PSTATE；
- CS_QUERY_FENCE_STATUS。

这保留了较多 amdgpu native 对象语义，但也意味着 Guest/Host ABI 强耦合。

## 6. Context 和 queue

### 6.1 Capset

Guest 读取 `virtio-gpu` capset 6。Capset 中提供：

- context type；
- alignment；
- Host UAPI/硬件信息；
- 后端能力。

Guest 用 `CONTEXT_INIT` 选择 `VIRTGPU_DRM_CAPSET_DRM`，AMD 实现通常使用
多个 virtio completion ring。

### 6.2 Context type

Host 根据 capset/context type 选择 backend，例如：

- MSM；
- AMDGPU；
- i915；
- Asahi；
- Panfrost。

Capset 6 是协议家族；Host 上的具体 native backend 还要匹配 Host GPU 和
driver。

### 6.3 Native context

Host backend 为 Guest application/context 建立 native device/context、timeline
和 ring/fence 映射。Guest native context ID、Host driver context ID 和 virtio
context ID 不是同一个编号。

## 7. CS 数据面

Guest native submit 通常包含：

- IB/command buffer；
- BO list；
- dependencies；
- in/out fence；
- syncobj/timeline；
- IP type/ring。

Guest shim 把这些 native chunks 打包进 `CS_SUBMIT` payload，并把这些 resource
通过 execbuffer BO list 传给 KMD。实际数据面因此有两部分：

1. control shmem：对象查询、创建和状态；
2. execbuffer/ring：CS command 和 native fence。

不能把 control request 当成 GPU command buffer，也不能把 virtio ring index
直接当成 Host native engine。

## 8. 内存模型

### 8.1 Host shmem

控制 shmem 是 `HOST3D + MAPPABLE` blob。Guest 映射的是 Host renderer 创建的
小共享窗口。

### 8.2 Guest resource

Guest BO 由 virtio-gpu resource/blob 表示。Native shim 维护：

```text
Guest libdrm-like handle
  -> virtio GEM handle
  -> virtio resource ID
  -> Host native BO
```

### 8.3 dma-buf

可共享资源可通过 dma-buf：

- Guest resource 导出 fd；
- Host native backend import 为 native BO；
- Host BO 也可 export 为 dma-buf 返回 Guest。

fd 传输解决 backing sharing，仍需处理：

- format/modifier；
- cache/coherence；
- ownership；
- pending GPU access；
- close/unref 顺序。

### 8.4 GPU VA

Guest native UMD/Guest shim 维护的 GPU VA 必须能被 Host native command 理解。
协议会把 BO VA op 映射为 Host native BO VA 操作。Guest VA 和 Host GPU VA
不能简单认为相同。

## 9. Fence 与 completion

有两到三层 completion：

### 9.1 Control shmem seqno

Host 处理 control request 后更新共享 `seqno`。它表示 request 已被 Host
处理，不表示 GPU work 完成。

### 9.2 virtio fence

Guest `VIRTGPU_EXECBUFFER` 可带 out fence FD。KMD/ virtio fence 完成表示
该次 virtio submit 的 completion。

### 9.3 Native GPU fence

Host backend 从 native submit 得到 dma-fence FD，并把 Guest ring/timeline
fence ID 与 native timeline 关联。ring 0 可用于 CPU/host sync；其他 ring 由
Host native fence retire。

三层必须分别匹配：

```text
control seqno != virtio execbuffer fence != native GPU completion
```

## 10. Resource 与 display

Native renderer 的 resource 仍挂在 virtio resource namespace 下。显示路径可
复用：

- resource/blob；
- dma-buf；
- `SET_SCANOUT_BLOB`；
- `RESOURCE_FLUSH`；
- QEMU display backend。

渲染/提交正确和 scanout 正确仍是两个验收面。

## 11. 为什么它不是 VFIO

| 维度 | Native DRM context | VFIO |
| --- | --- | --- |
| Guest 物理 BAR | 无 | 有 |
| Guest 原生 KMD | 否，virtio-gpu KMD | 是 physical GPU driver |
| Host command 解析 | native renderer/proxy | 通常无 |
| IOMMU group | Host renderer 进程所属 | Guest 设备所属 |
| reset | 虚拟设备/Host renderer | 物理 FLR/device reset |
| migration | renderer state 约束 | VFIO migration UAPI |
| 隔离 | 软件 parser + Host driver | IOMMU + PCIe function |

它可能比高层 API remoting 保留更多 driver 语义，但不具备 VFIO 的物理设备
边界。

## 12. 与 VirGL/Venus/GFXStream 的区别

| 路径 | Guest remote contract |
| --- | --- |
| VirGL | Gallium/OpenGL commands |
| Venus | Vulkan wire protocol |
| GFXStream | GFXStream API command stream |
| Native DRM | vendor DRM UAPI-like control/CS |
| VFIO | physical driver + PCI/MMIO |

Native context 不要求 Host 把 command 解释成 GL/Vulkan API；但 Guest shim 必须
生成 Host backend 期望的 native UAPI model。

## 13. 优点

- 保留更多 native driver 对象和 command 语义；
- 潜在减少高层 API state translation；
- 可复用原生 barrier/fence/VM 模型；
- 适合图形/原生 DRM 研究；
- 可比较“API remoting”和“driver UAPI remoting”。

## 14. 限制

- Guest/Host driver UAPI 强耦合；
- backend 依厂商和 GPU；
- resource、VA、dma-buf 语义复杂；
- Host renderer 是安全边界；
- 不自动支持 CUDA/HIP；
- 不支持透明 live migration；
- 需要 Host EGL/render node/driver 条件；
- Guest UMD 不是任意物理 GPU driver 二进制原样运行。

## 15. 生命周期和故障

### 15.1 Init

必须成功：

- capset；
- context type；
- Host backend probe；
- control shmem；
- resource table；
- native device/context。

### 15.2 Reset

Host/guest context、native GPU VM、resources、timeline 和 pending fences
都要处理。Native context 不是只丢 command bytes 就能恢复。

### 15.3 Migration

Host native renderer state、native driver object、dma-buf/fd、GPU VM 和 timeline
使 live migration 很难。不能从 virtio 2D VMState 外推。

## 16. 何时研究/使用

适合：

- 想理解 driver UAPI remoting；
- Host/guest GPU 软件栈可控；
- 图形或受控 native workload；
- 需要比高层 API 更贴近 driver 的路径。

不适合：

- 需要无修改 CUDA/HIP；
- 需要强 PCIe 硬件隔离；
- 需要跨厂商 arbitrary driver；
- 需要透明迁移；
- 无法锁定 Guest/Host 软件版本。

## 17. 检查题

1. Native DRM context 为什么不是 VFIO？
2. Guest UMD、vdrm、virtio-gpu KMD 和 Host renderer 各负责什么？
3. control shmem 与 CS command buffer 有什么区别？
4. 为什么 response 需要 shadow buffer？
5. Guest handle、resource ID 和 Host native BO 为什么不是一回事？
6. ring 0、native timeline 和 virtio fence 如何对应？
7. Guest GPU VA 为什么不能直接交给 Host native command？
8. Native context 为什么仍受 Guest/Host driver version 强约束？
9. 为什么它不自动支持 ROCm/CUDA？
10. 为什么 renderer 的 native UAPI 支持不等于完整 VM live migration？
