# GPU 虚拟化专家训练工作簿

## 1. 使用方式

这份 workbook 把现有文档转成可检查的输出。当前不运行 VM、GPU、benchmark、
迁移或故障实验；练习只要求读源码、画状态机、写评审和设计验证计划。

建议每个练习产出：

1. 一页调用链或状态机图；
2. 一张对象所有权表；
3. 至少一个失败路径；
4. 一个“不能从现有证据推出什么”的边界说明；
5. 对应源码链接和固定 commit。

完成后用每节的 acceptance checklist 自评。

## 2. 训练基线

先固定：

| 项目 | Revision |
| --- | --- |
| Linux | `6b4b543fa18757297ac756a6fe6e64f506b7844a` |
| Mesa | `ec7b2f322abbcd593d600a14afe2d2773512af4e` |
| virglrenderer | `e0a1caf3b147bb26dfe524dbb85ded1177e7e49e` |
| QEMU | `d73149283205645625bfa9fe029cfb68e4c4a9ae` |

证据等级和状态词按
[EVIDENCE_REGISTER.md](../EVIDENCE_REGISTER.md) 使用。

## 3. Exercise 1：一次 VirGL draw 的完整源码追踪

### 目标

从 Guest GL draw 追到 Host GL dispatch，并准确说出每个边界。

### 必读

- [VirGL 宏观](../virtio-gpu/virgl-overview.md)
- [VirGL 源码跟踪](../virtio-gpu/virgl-source-trace.md)
- [Vhost-user-gpu 外置后端](../virtio-gpu/vhost-user-gpu-source-trace.md)
- `mesa/src/gallium/drivers/virgl/virgl_context.c`
- `mesa/src/gallium/drivers/virgl/virgl_encode.c`
- `mesa/src/gallium/winsys/virgl/drm/virgl_drm_winsys.c`
- `linux/drivers/gpu/drm/virtio/virtgpu_submit.c`
- `linux/drivers/gpu/drm/virtio/virtgpu_vq.c`
- `qemu/hw/display/virtio-gpu-virgl.c`
- `virglrenderer/src/vrend/vrend_decode.c`
- `virglrenderer/src/vrend/vrend_renderer.c`

### 任务

1. 标出 `virgl_draw_vbo()` 到 `glDraw*()` 的每次函数跳转。
2. 写明 Meson command buffer flush 的触发条件。
3. 标出 Guest command buffer、input fence、output fence、BO list 的所有权。
4. 画出 command response 和 GPU fence completion 两条不同路径。
5. 指出 QEMU 在什么条件下 suspend command。

###  Acceptance

- [ ] 能解释为什么 Guest KMD 不解析 draw opcode。
- [ ] 能指出一次 draw 可能不导致一次 virtio submit。
- [ ] 能说明 `RESOURCE_FLUSH` 与 draw fence 的区别。
- [ ] 能说明 BO list 防止什么生命周期问题。
- [ ] 能说明进程内 renderer 与外置 vhost-user renderer 的边界差异。

## 4. Exercise 2：Virtio-GPU command parser 安全评审

### 必读

- `qemu/hw/display/virtio-gpu.c`
- `qemu/hw/display/virtio-gpu-virgl.c`
- `qemu/hw/display/virtio-gpu-rutabaga.c`
- `virglrenderer/src/virglrenderer.c`
- [安全与故障域](gpu-virtualization-security.md)

### 任务

1. 列出 Guest 可控制的长度、offset、ID、alignment 和 handle。
2. 为每个字段找到至少一层验证。
3. 找出“存在性检查”与“所有权检查”的差异。
4. 分析非法 command 后的 response、资源释放和 fence 状态。
5. 检查 response buffer 写入是否可从 Guest 输入越界。

### Acceptance

- [ ] 不把“renderer 有 Rust/C 类型”当作内存安全检查。
- [ ] 能说明每层为何应重复验证自己依赖的不变量。
- [ ] 能指出至少一个 TOCTOU 风险。
- [ ] 能给出资源创建失败的 cleanup 顺序。

## 5. Exercise 3：Resource/blob 生命周期

### 必读

- `linux/drivers/gpu/drm/virtio/virtgpu_object.c`
- `qemu/hw/display/virtio-gpu.c`
- `qemu/hw/display/virtio-gpu-virgl.c`
- `virglrenderer/src/virglrenderer.c`
- [GPU 显存语义](gpu-memory-semantics.md)

### 任务

1. 画 Guest GEM -> resource ID -> Host renderer resource -> backing 的所有权图。
2. 对 `GUEST/HOST3D/HOST3D_GUEST` 三种 blob 各画一条数据路径。
3. 对 create、attach、map、submit、complete、unmap、unref 标出 fence 依赖。
4. 列出失败窗口：attach 成功但 renderer create 失败；map 成功但 response 失败。
5. 判断哪种路径可能真正 dma-buf 共享，哪种只是 staging。

### Acceptance

- [ ] 不把 guest page 等同于 Host GPU backing。
- [ ] 不把 resource ID 等同于 Host pointer。
- [ ] 能解释 unmap 与 pending command 的 race。
- [ ] 能说明 map_info/cache 不是 DMA coherence 的完整证明。

## 6. Exercise 4：Venus ring 与 fence

### 必读

- [Venus 宏观](../virtio-gpu/venus-overview.md)
- [Venus 源码跟踪](../virtio-gpu/venus-source-trace.md)
- `mesa/src/virtio/vulkan/vn_ring.c`
- `virglrenderer/src/proxy/proxy_context.c`
- `virglrenderer/src/venus/vkr_ring.c`
- `virglrenderer/src/venus/vkr_context.c`

### 任务

1. 画出 ring 内存布局和 head/tail/status 写者。
2. 写出 tail release、head acquire 的 happens-before 关系。
3. 写出大 command upload 路径，以及需要保活的 shmem/object。
4. 区分 Host consumed seqno、VkFence、present fence。
5. 找出 notify 丢失或 head 不前进时的检查顺序。

### Acceptance

- [ ] 能解释 ring head 前移不等于 GPU work 完成。
- [ ] 能说明 proxy render server 断开后的 fence 结果。
- [ ] 能指出 Venus object ID 与 Host Vulkan handle 不同。

## 7. Exercise 5：GFXStream/Rutabaga 边界

### 必读

- [GFXStream 宏观](../virtio-gpu/gfxstream-overview.md)
- [GFXStream 源码跟踪](../virtio-gpu/gfxstream-source-trace.md)
- [Rutabaga 宏观](../virtio-gpu/rutabaga-overview.md)
- [Rutabaga 源码跟踪](../virtio-gpu/rutabaga-source-trace.md)
- `qemu/hw/display/virtio-gpu-rutabaga.c`
- `rutabaga_gfx_ffi-0.1.3/src/lib.rs`
- `rutabaga_gfx-0.1.3/src/rutabaga_core.rs`
- `rutabaga_gfx-0.1.3/src/gfxstream.rs`

### 任务

1. 区分 C ABI、Rust core、RutabagaComponent、GFXStream backend。
2. 画 ASG ring、QEMU submit 和 fence callback 的线程边界。
3. 分析 `catch_unwind()` 能/不能防住的错误。
4. 解释为什么 snapshot API 不等于 QEMU migration。
5. 设计一个不依赖 performance test 的 component 接口审查表。

### Acceptance

- [ ] 能说明 Rutabaga 不是 Guest UMD。
- [ ] 能解释 opaque pointer/长度/所有权 contract。
- [ ] 能指出 QEMU AIO bottom half 的原因。
- [ ] 能区分 Cargo version mismatch 和 GPU ABI mismatch。

## 8. Exercise 6：Native DRM context / AMDGPU shim

### 必读

- [Native DRM 宏观](../virtio-gpu/native-drm-overview.md)
- [Native DRM 源码跟踪](../virtio-gpu/native-drm-source-trace.md)

### 任务

1. 从 `amdvgpu_cs_submit_raw2()` 追到 Host `amdgpu_cs_submit_raw2()`。
2. 列出 Guest chunks、descriptors、BO list、syncobj 如何转换。
3. 画 control shmem 和 CS command data 两条通道。
4. 找出 Host response shadow buffer 和 double-fetch 防护。
5. 解释 ring 0 与 native GPU fence 的差别。

### Acceptance

- [ ] 能说明 native context 不是 VFIO。
- [ ] 能说明 control seqno 不是 GPU completion。
- [ ] 能指出 Guest/Host driver ABI 的耦合点。
- [ ] 能指出 dma-buf import/export 的生命周期风险。

## 9. Exercise 7：GPU VM/BO/HMM/SVM

### 必读

- [GPU 显存语义](gpu-memory-semantics.md)
- [DMA-BUF/DMA-FENCE 源码跟踪](dma-buf-fence-source-trace.md)
- [DRM Syncobj 源码跟踪](drm-syncobj-source-trace.md)
- [DRM GEM/PRIME 源码跟踪](drm-gem-prime-source-trace.md)
- [Intel Xe SVM 源码跟踪](intel-xe-svm-source-trace.md)
- `linux/drivers/gpu/drm/drm_gpusvm.c`
- `linux/drivers/gpu/drm/drm_pagemap.c`
- `linux/drivers/gpu/drm/amd/amdgpu/amdgpu_hmm.c`
- `linux/drivers/gpu/drm/amd/amdkfd/kfd_svm.c`

### 任务

1. 区分 GPU VA、IOVA、GPA、Host VA、HPA 和 VRAM offset。
2. 画 `mmu_interval_read_begin -> hmm_range_fault -> retry` 的竞态窗口。
3. 画 system -> VRAM migration 和 eviction 回 RAM。
4. 定位 GPU PTE update、TLB invalidate、page unlock/finalize 的顺序。
5. 解释为什么 physical page 不能保存可变的 CPU VA backpointer。

### Acceptance

- [ ] 能说明 DMA map 成功不等于 coherent。
- [ ] 能说明 mapping、residency、pin、lifetime 是不同维度。
- [ ] 能解释 migration retry/livelock 来源。
- [ ] 能说明 `migrate_device_*` eviction 为什么可不持有 mmap lock。
- [ ] 能解释 Xe fault storm chaining 与 userptr invalidation 的顺序。
- [ ] 能区分 implicit/explicit fence 以及 dma-buf fd 不携带的 metadata。
- [ ] 能解释 virtio execbuffer input wait 与 output syncobj completion。
- [ ] 能解释 GEM handle、fake mmap offset 和 dma-buf fd 的 namespace 差异。

## 10. Exercise 8：KFD AQL launch 源码状态机

### 必读

- [AMD KFD/ROCr 源码跟踪](amdkfd-source-trace.md)
- `linux/drivers/gpu/drm/amd/amdkfd/kfd_chardev.c`
- `kfd_process_queue_manager.c`
- `kfd_queue.c`
- `kfd_doorbell.c`
- `kfd_events.c`
- `Documentation/gpu/amdgpu/userq.rst`

### 任务

1. 画 AQL ring、rptr/wptr、doorbell、MQD/HQD、firmware scheduler。
2. 分析 queue oversubscription 与 aggregate doorbell。
3. 画 event signal -> IRQ/PASID -> waitqueue。
4. 分析 migration 时 queue preempt/migrate/PTE/TLB/resume。
5. 区分 per-queue reset、engine reset、whole GPU reset。

### Acceptance

- [ ] 能说明 doorbell 写不等于 kernel 完成。
- [ ] 能解释 queue priority 不是绝对资源上限。
- [ ] 能说明 long compute 无法只靠 dma-fence 管理。
- [ ] 能指出 process exit 时 fault/queue/event teardown 顺序。

## 11. Exercise 9：Userptr/ROCm RFC 评审

### 必读

- [Virtio-GPU Userptr 与 ROCm](../virtio-gpu/userptr-rocm-compute.md)
- [ROCm compute 源码跟踪](../virtio-gpu/rocm-source-trace.md)
- [AI Runtime/API 兼容](ai-runtime-api-compatibility.md)

### 任务

按组件列 matrix：

```text
Linux UAPI
QEMU wire
virglrenderer protocol
ROCm frontend
Host libhsakmt/KFD
```

逐项检查：

- capset/flag/version；
- UAPI field/feature；
- pin/DMA coherence；
- cleanup；
- map_info；
- fence/multi-event；
- dma-buf；
- migration。

### Acceptance

- [ ] 不把 RFC v9 当 merged/released。
- [ ] 能指出 kernel v9 与 QEMU v4 的错位。
- [ ] 能指出 capset 8/10 和 bit 14 冲突。
- [ ] 能说明 HIP coarse/fine coherence 对平台的要求。
- [ ] 能列出至少三个 release gate。

## 12. Exercise 10：VFIO/SR-IOV 故障树

### 必读

- [平台比较](../virtio-gpu/platform-comparison.md)
- [ATS/PASID/PRI/IOMMU SVA 源码跟踪](ats-pasid-pri-source-trace.md)
- [VFIO/IOMMUFD/SR-IOV/mdev 源码跟踪](vfio-iommufd-sriov-source-trace.md)
- [QEMU VFIO 源码跟踪](qemu-vfio-source-trace.md)
- [NVIDIA Grace GPU VFIO Variant 源码跟踪](nvidia-grace-vfio-source-trace.md)
- [安全与故障域](gpu-virtualization-security.md)
- [Intel Xe SR-IOV](intel-xe-sriov.md)
- [Intel GVT-g 源码跟踪](intel-gvt-g-source-trace.md)
- [AMD GPU SR-IOV](amd-gpu-sriov-mxgpu.md)
- `Documentation/driver-api/vfio.rst`
- `Documentation/PCI/pci-iov-howto.rst`
- `Documentation/PCI/pcieaer-howto.rst`

### 任务

设计故障树：

```text
VF 不出现
  -> PCI cap
  -> PF readiness/driver
  -> provisioning
  -> VF enable
  -> sysfs/device

VF 出现但 DMA/queue 错
  -> IOMMU
  -> VF BAR/resource
  -> firmware config
  -> FLR

一个 VF reset 影响别人
  -> shared firmware/scheduler
  -> reset scope
  -> PF management
```

### Acceptance

- [ ] 能解释 IOMMU group 为什么可能大于一个设备。
- [ ] 能区分 legacy group/container 与 device cdev/IOMMUFD。
- [ ] 能说明 VF 不等于完全独立 GPU。
- [ ] 能区分 VF stop、FLR、PCI AER。
- [ ] 能说明 IOMMU 不是 command parser 安全边界。
- [ ] 能区分 GVT-g parser/shadow 与 SR-IOV hardware VF。
- [ ] 能解释 ATS ATC、IOMMU TLB、GPU TLB 和 CPU TLB 的失效边界。

## 13. Exercise 11：AI workload 架构选型

### 必读

- [AI GPU 计算栈](ai-compute-virtualization.md)
- [OpenCL/Level Zero/SYCL 边界](opencl-levelzero-sycl-virtualization.md)
- [架构 Playbook](gpu-virtualization-design-playbook.md)
- [云原生 GPU](cloud-native-gpu-sharing.md)
- [NVIDIA MIG/vGPU](nvidia-cuda-vgpu.md)
- [Windows/Hyper-V GPU](windows-hyperv-gpu-virtualization.md)
- [NCCL/RCCL](nccl-rccl-source-trace.md)

### 场景

A. 单租 70B 模型训练；
B. 多租 7B 推理，要求 P99；
C. CI 中 Android 图形；
D. 跨厂商 Linux 图形；
E. 现有 HIP 代码不改；
F. 推理 decode 小 kernel。

### 任务

为每个场景输出：

1. 候选机制；
2. hard gates；
3. 失败模式；
4. 最小证据；
5. rollout/rollback；
6. 明确拒绝的方案及原因。

### Acceptance

- [ ] 不因单卡 GEMM 接近原生而选择所有场景。
- [ ] 不把 MIG/vGPU/VF 视为同层。
- [ ] 能把 CUDA/HIP ABI 作为 hard gate。
- [ ] 能区分 OpenCL translation、SYCL backend 选择和 Level Zero proxy。
- [ ] 能区分 DDA、GPU-P 和 WSL2 GPU-PV 的设备/API 边界。

## 14. Exercise 12：迁移与 checkpoint 审查

### 必读

- [GPU VM 生命周期、Checkpoint 与迁移](vm-lifecycle-migration.md)

### 任务

选择 Virtio-GPU VirGL、Rutabaga、VFIO、Intel VF、AMD VF 中任意三种，分别填：

```text
Guest ABI
Virtqueue/PCI state
Host object
GPU VM/PTE
VRAM/context
Fence/event
Display/external fd
Source/destination compatibility
Rollback/double-owner prevention
```

### Acceptance

- [ ] 区分 reset/suspend/snapshot/CRIU/live migration。
- [ ] 不因 VMState 或 snapshot API 直接宣称 live migration。
- [ ] 能指出 destination 接管前 source 必须保留恢复状态。

## 15. Exercise 13：性能 trace 设计

### 必读

- [性能与可观测性](gpu-performance-observability.md)
- `PERFORMANCE_AND_HARDWARE_BASELINE.md`

### 任务

为一次 decode 请求设计 trace：

```text
request
guest runtime
guest submit
virtqueue
qemu
renderer/host runtime
GPU queue
GPU completion
guest fence
response
```

标记：

- clock domain；
- correlation ID；
- success/failure；
- P99 关注段；
- instrumentation overhead；
- 数据脱敏。

### Acceptance

- [ ] 不使用平均 GPU utilization 替代端到端延迟。
- [ ] 不把 API 返回时间当 GPU completion。
- [ ] 能区分 launch-bound/HBM-bound/sync-bound。

## 16. Exercise 14：安全威胁模型

### 必读

- [安全与故障域](gpu-virtualization-security.md)

### 任务

针对 Virtio-GPU renderer、VFIO VF、容器 MPS 各写一个 STRIDE 风格表：

- asset；
- trust boundary；
- attacker input；
- failure impact；
- prevention；
- detection；
- recovery；
- residual risk。

### Acceptance

- [ ] 不把 IOMMU、Rust、MIG 或 device plugin 当万能安全证明。
- [ ] 覆盖数据残留、side channel、reset 和 management plane。
- [ ] 对 unknown 使用 unknown，而不是推测为安全。

## 17. Exercise 15：Capstone 架构评审

### 场景

设计一个支持以下需求的 GPU 云：

- 50% 单租训练；
- 30% 多租推理；
- 20% 图形/VDI；
- Linux 和 Windows Guest；
- 基本 HIP/CUDA 兼容；
- 可选 live migration；
- 严格租户隔离；
- GPU 型号异构。

### 任务

输出完整提案：

1. 资源池和 GPU 型号；
2. device exposure 组合；
3. API/ABI support matrix；
4. memory/QoS；
5. security；
6. lifecycle/upgrade；
7. observability；
8. capacity planning；
9. failure/recovery；
10. phased rollout；
11. unsupported claims；
12. open questions。

### Acceptance

- [ ] 不假设一个机制覆盖所有 workload。
- [ ] 每个候选都有 hard gate 和证据等级。
- [ ] 迁移不是默认能力。
- [ ] 结论能经受“如果硬件/驱动/profile 变化”审计。

## 18. 自评等级

### Level 1：能解释

- 看懂宏观文档；
- 能画出单路径；
- 能复述术语。

### Level 2：能追踪

- 能从 API 追到源码函数；
- 能区分 control/data/completion；
- 能定位失败层。

### Level 3：能比较

- 能比较 virtio/VFIO/SR-IOV/MIG/vGPU；
- 能说明 ABI/coherence/lifecycle 边界；
- 能发现 capability overclaim。

### Level 4：能设计

- 能基于 workload 做方案选择；
- 能设计证据矩阵和 rollout；
- 能写出可执行的安全/failure review；
- 能识别尚缺的源码/实验证据。

达到 Level 4 后，下一步应选择真实平台做受控实验，用实验证据替代 unknown。

## 19. 推荐顺序

```text
1-3   四路径源码和 virtio transport
4-6   resource/fence/Venus
7-8   GFXStream/Rutabaga/native DRM
9-10  GPU VM/HMM/SVM/KFD
11    userptr/ROCm
12    VFIO/SR-IOV 厂商路线
13    AI workload/API compatibility
14    lifecycle/migration
15    performance/security
16    capstone
```

这套训练与 [学习路线](learning-roadmap.md) 配套：路线负责顺序，本文负责可验收输出。
