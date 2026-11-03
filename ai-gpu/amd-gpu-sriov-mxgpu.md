# AMD GPU SR-IOV 与 MxGPU 源码视角

## 1. 文档定位

本文补 AMD GPU 的 SR-IOV/MxGPU 路径。它不是 virtio-gpu API remoting：

- Guest 运行 AMD Guest GPU driver；
- Guest 看到 PCI VF；
- PF/Host hypervisor 侧管理 VF 资源；
- Guest 通过 VFIO/vendor platform 获得设备；
- Host 仍由 AMD PF driver、firmware 和 hypervisor/vGPU manager 管理。

AMD MxGPU 是 AMD 的 SR-IOV GPU 虚拟化方案。本文只使用固定 Linux
`amdgpu`/KFD 公开源码解释其 Guest/Host 驱动边界，不把未公开 hypervisor/
firmware 内部实现写成源码事实。

源码基线：Linux `6b4b543fa18757297ac756a6fe6e64f506b7844a`。

本文不运行 AMD GPU、VF、VM、ROCm 或迁移实验。

## 2. 与 virtio-gpu ROCm proposal 的区别

| 维度 | AMD SR-IOV/MxGPU | virtio-gpu ROCm proposal |
| --- | --- | --- |
| Guest 设备 | PCI VF | virtio-gpu + ROCm capset |
| Guest driver | amdgpu + KFD VF path | HSAKMT/ROCm virtio frontend |
| Host execution | PF/GPU firmware/hypervisor | Host renderer + libhsakmt/KFD |
| PCI BAR | VF BAR | 无物理 GPU BAR |
| Host command parsing | Guest commands mostly remain native | Host renderer parses proxied objects |
| Current status | 多个 AMD 平台实现 | RFC/MR，未稳定 |

SR-IOV 不是 API remoting；ROCm proposal 也不是 AMD SR-IOV 的替代协议。

## 3. Guest/Host 架构

```text
Physical AMD GPU
  -> PF driver / platform manager / firmware
  -> SR-IOV VF
  -> VFIO/vendor assignment
  -> Guest amdgpu DRM + KFD
  -> Guest graphics/HIP/ROCr
  -> VF MMIO/queue/doorbell/interrupt
  -> physical GPU scheduler
```

Guest compute 仍使用：

- `/dev/dri/renderD*`：amdgpu DRM；
- `/dev/kfd`：KFD/HSA compute；
- AQL/SDMA queues；
- event、GPU VM、VRAM/GTT/userptr。

因此它更容易保留 ROCm/HIP/native runtime，但也继承 VF/profile/firmware/
reset 的支持矩阵。

## 4. SR-IOV/Passthrough 检测

### 4.1 Caps

[`amdgpu_virt.h`](../../linux/drivers/gpu/drm/amd/amdgpu/amdgpu_virt.h:29)
定义：

- `SRIOV_VBIOS`：vBIOS SR-IOV ready；
- `ENABLE_IOV`：当前 GPU 启用 SR-IOV；
- `IS_VF`：当前是 VF；
- `PASSTHROUGH_MODE`：整卡 passthrough；
- `SRIOV_CAPS_RUNTIME`：VF 处于非 full-access 运行态；
- `VF_MMIO_ACCESS_PROTECT`：runtime 下多数 CPU MMIO write 被阻止；
- `VIRT_VRAM_IS_SPM`：FB BAR0 缺失，VRAM 由 SPM 抽象。

这些 caps 是后续 driver 分支的关键。

### 4.2 Register-based detection

[`amdgpu_virt_init_detect_asic()`](../../linux/drivers/gpu/drm/amd/amdgpu/amdgpu_virt.c)
根据 ASIC 读取 IOV function identifier：

- Tonga/Fiji 使用 BIF register；
- 新 ASIC 使用 RCC register；
- bit 0 表示 VF；
- 高位表示 IOV enabled；
- register 0 且处于 VM 时进入 passthrough mode。

驱动不是只根据 PCI vendor string判断。

## 5. Full access 与 runtime

### 5.1 初始化需要 full access

VF driver 在 init/fini 阶段通过 mailbox 请求 full GPU access：

- `amdgpu_virt_request_full_gpu()`
- `amdgpu_virt_release_full_gpu()`

成功后清除 `SRIOV_CAPS_RUNTIME`；release 后恢复 runtime。

### 5.2 Runtime 保护

runtime 时：

- 多数 MMIO 页除 mailbox 外可能被 hypervisor 阻断；
- `amdgpu_virt_mmio_blocked()` 通过 scratch register 判断；
- 某些 IP 寄存器通过 PSP/RLC/RLCG 间接访问；
- VF 不能假设和 bare metal 一样直接访问所有寄存器。

这是 MxGPU 与 passthrough 的重要差别：Guest driver 仍 native，但部分硬件
寄存器被 mux/protect。

## 6. Mailbox 控制面

### 6.1 VF/PF mailbox

AI/VI/NV 平台各有实现，公开路径：

- `mxgpu_ai.c`
- `mxgpu_vi.c`
- `mxgpu_nv.c`

消息通过 BIF/NBIO mailbox register 和 interrupt 传输。

[`mxgpu_ai.c`](../../linux/drivers/gpu/drm/amd/amdgpu/mxgpu_ai.c:37) 实现：

- mailbox valid/ack；
- poll ACK/msg；
- transmit request；
- receive interrupt；
- ack interrupt；
- FLR work；
- bad page/RAS work。

### 6.2 典型 request

公开 `idh_request` 包括：

- `REQ_GPU_INIT_ACCESS`；
- `REL_GPU_INIT_ACCESS`；
- `REQ_GPU_FINI_ACCESS`；
- `REL_GPU_FINI_ACCESS`；
- `REQ_GPU_RESET_ACCESS`；
- `REQ_GPU_INIT_DATA`；
- `LOG_VF_ERROR`；
- `READY_TO_RESET`；
- RAS poison/bad-page request。

公开 event 包括：

- `READY_TO_ACCESS_GPU`；
- `FLR_NOTIFICATION`；
- `FLR_NOTIFICATION_CMPL`；
- `FAIL`；
- `RAS_ERROR_DETECTED`；
- `UNRECOV_ERR_NOTIFICATION`。

### 6.3 Mailbox 不等于 data plane

mailbox 主要承载：

- access request；
- reset/FLR；
- init data；
- error/RAS 通知。

它不承载每个 graphics/HIP work 的执行命令。

## 7. Init data 与 critical regions

### 7.1 PF2VF/VF2PF 共享区

[`amdgpu_virt.h`](../../linux/drivers/gpu/drm/amd/amdgpu/amdgpu_virt.h:137)
定义 PF/VF 信息结构与 feature flags，例如：

- GIM load ucode；
- FLR 后 VRAM lost；
- MM bandwidth management；
- one-VF mode；
- indirect register access；
- MES info；
- RAS capabilities/telemetry；
- XGMI peer/CPU link；
- PTL。

### 7.2 Dynamic critical region

`INDA` signature/table 描述：

- IP discovery；
- VBIOS image；
- RAS telemetry；
- data exchange；
- bad page info；
- init header。

VF 在 init 时从 PF/firmware 提供的区域取得必要元数据，并创建保留 VRAM
region。该区域不是提供给 Guest 应用的普通 VRAM。

### 7.3 VF2PF telemetry

VF2PF structure 可上报：

- driver version/certification；
- guest OS；
- FB usage；
- graphics/compute usage/health；
- VCE/UVD usage/health。

它是管理/遥测契约，必须按 PF/VF 版本验证，不能当作任意数值都可信。

## 8. Indirect register access

### 8.1 需求

runtime VF 的多数 MMIO write 被保护。某些 IP 仍需要初始化/状态访问。

`amdgpu_virt.h` 提供：

- PSP 编程 IH；
- RLC 编程 MMHUB；
- RLC 编程 GC；
- PSP 编程 L1 TLB control；
- RLCG 编程 SQ config。

### 8.2 RLCG/VFi

[`amdgpu_virt.c`](../../linux/drivers/gpu/drm/amd/amdgpu/amdgpu_virt.c) 中
indirect path：

- 检查 VFi availability；
- 写 command/address/data scratch；
- poll status；
- 区分 done/invalid cmd/invalid address/timeout/error；
- 对 VF-gate disabled 等错误报告。

这说明 “Guest native driver” 仍可能通过 firmware-mediated register path。

## 9. Queue、VM 与内存

### 9.1 KFD queue

Guest ROCm/HIP 仍通过 KFD 创建 AQL/SDMA queue，使用：

- ring/rptr/wptr；
- doorbell；
- EOP/context save；
- PASID/GPU VM。

VF PF/hypervisor 负责给该 VF 分配可见资源；具体可见 VMID/queue/doorbell
范围依 ASIC/platform。

详见 [AMD KFD/ROCr 源码跟踪](amdkfd-source-trace.md)。

### 9.2 GPU VM

Guest amdgpu 建立 Guest GPU VM/PTE。physical GPU 访问 Guest system memory
仍经 Host IOMMU。

“VF 内 native GPU VA”不代表绕过 Host IOMMU。

### 9.3 VRAM

VF 可能：

- 有 VF FB/VISIBLE VRAM aperture；
- FB BAR0 缺失、通过 SPM 机制访问；
- 收到 `GIM_FLR_VRAMLOST`，表示 FLR 后 VRAM 内容丢失。

因此应用/驱动不能假设 reset 后 VRAM 一定保留。

## 10. Reset 与 FLR

### 10.1 Host FLR

[`amdgpu_device_reset_sriov()`](../../linux/drivers/gpu/drm/amd/amdgpu/amdgpu_device.c:4839)
在 Host FLR 路径：

1. 非 fatal error 时发送 `READY_TO_RESET`；
2. 等待 Host reset completion；
3. 请求 full GPU access；
4. clear RAS error state；
5. post-reset 软件修正；
6. early IP reinit；
7. re-init data exchange；
8. reload firmware；
9. late IP reinit SMC/CP/SDMA；
10. 更新 XGMI topology；
11. IB ring tests；
12. 按 `VRAMLOST` 增加 lost count。

### 10.2 Self reset

其他路径发送 `REQ_GPU_RESET_ACCESS` 给 PF/hypervisor，然后走类似 reinit。

### 10.3 One-VF/multi-VF

driver 区分：

- `SRIOV_VF_MODE_ONE_VF`；
- `SRIOV_VF_MODE_MULTI_VF`；
- bare metal。

one-VF mode 的性能/独占策略可能与 multi-VF 不同，不能直接外推。

## 11. Error 与 RAS

### 11.1 VF error

[`amdgpu_vf_error.c`](../../linux/drivers/gpu/drm/amd/amdgpu/amdgpu_vf_error.c)
用 ring buffer 保存 VF error，并通过 `trans_msg(IDH_LOG_VF_ERROR, ...)`
发给 PF。

### 11.2 RAS

如果 GIM feature 支持：

- RAS error notification；
- telemetry；
- bad pages；
- CPER/poison；
- critical region；
- PTL。

VF 可读取/上报 RAS 信息，但支持集合由 PF/hypervisor feature flags 决定。

### 11.3 Unrecoverable

`UNRECOV_ERR_NOTIFICATION` 可让 runtime services 停止。产品必须定义：

- 是否只影响 VF；
- 是否要求 VF FLR；
- 是否影响 sibling VF；
- checkpoint/重试策略。

## 12. Suspend/hibernate

Xe 和 AMD 的 suspend 处理不同，但共同问题都是：

- VRAM 可能掉电；
- queue/context/MQD 需要 eviction/restore；
- firmware/data exchange 需要重建；
- PF/VF resume 顺序；
- Guest event/fence 状态。

AMD `VRAMLOST` flag 明确说明 FLR 与 VRAM 保留不是同一件事。

## 13. VFIO/VM 路径

```text
AMD PF / hypervisor manager
  -> VF
  -> VFIO
  -> VM
  -> Guest amdgpu + KFD
  -> ROCm/HIP/graphics
```

需要验证：

- VF PCI identity/BIOS；
- VF BAR/FB aperture；
- IOMMU/ACS；
- MSI-X；
- one-VF/multi-VF mode；
- reset/migration；
- XGMI/P2P；
- ROCm KFD topology；
- vBIOS/ucode visibility。

## 14. 安全边界

- VF driver 不可执行 privileged PF registers；
- mailbox 写入必须由 PF 验证；
- init data/critical region 长度、signature、version 和 checksum；
- VF2PF telemetry 不可被其他 VF 读取；
- bad-page/RAS 区域按 VF 隔离；
- RESET/FLR 确认旧 queue/DMA 已停止；
- IOMMU 限制 Guest system-memory DMA；
- firmware/hypervisor 故障可能影响 sibling VF。

Guest kernel 可信度不能替代 IOMMU/firmware 边界。

## 15. 性能关注

- one-VF vs multi-VF；
- VF time-slice/engine sharing；
- VRAM/bandwidth partition；
- MMIO protection/indirect register cost；
- mailbox latency；
- KFD queue scheduling；
- XGMI/P2P topology；
- FLR 后 VRAM lost/rebuild；
- telemetry/RAS overhead。

## 16. 与 Intel Xe/NVIDIA 对照

| 维度 | AMD MxGPU/SR-IOV | Intel Xe SR-IOV | NVIDIA MIG/vGPU |
| --- | --- | --- | --- |
| PCI level | VF | VF | vGPU/SR-IOV 依产品 |
| guest graphics driver | amdgpu | xe | NVIDIA guest driver |
| compute driver | KFD/ROCr | Xe compute UAPI | CUDA |
| host resource mgmt | PF/GIM/hypervisor/firmware | PF/GuC KLV | MIG partition/vGPU manager |
| reset | mailbox+FLR+IP reinit | VF control/FLR | product-specific |
| migration | 依 platform/GIM/产品 | PF/VF migration + VFIO | product dependent |

## 17. 检查题

1. AMD MxGPU 为什么不是 virtio-gpu API remoting？
2. VF 检测为什么需要 IOV function register？
3. `SRIOV_CAPS_RUNTIME` 与 full access 的关系是什么？
4. runtime 下 MMIO protection 为什么需要 PSP/RLC/RLCG indirect access？
5. mailbox 控制面承载哪些消息，不承载什么？
6. PF2VF/VF2PF 共享区与 critical region 的用途是什么？
7. one-VF 与 multi-VF 模式为什么可能影响性能/隔离？
8. FLR 后 `VRAMLOST` 对应用恢复意味着什么？
9. Host FLR 与 Guest self reset 路径有什么差异？
10. VF error/RAS 如何从 Guest 传回 PF？
11. VF native GPU VM 为什么仍受 Host IOMMU 约束？
12. AMD VF、Intel VF、NVIDIA MIG/vGPU 各在哪个抽象层？

## 18. 与其他专题

- KFD/ROCr queue/event/SVM：  
  [AMD KFD/ROCr 源码跟踪](amdkfd-source-trace.md)
- Virtio-GPU ROCm proposal：  
  [Virtio-GPU Userptr 与 ROCm](../virtio-gpu/userptr-rocm-compute.md)
- Intel Xe VF：  
  [Intel Xe GPU 虚拟化与 SR-IOV](intel-xe-sriov.md)
- NVIDIA MIG/vGPU：  
  [NVIDIA CUDA、MIG 与 vGPU](nvidia-cuda-vgpu.md)
- 平台与 VFIO：  
  [GPU 虚拟化平台比较](../virtio-gpu/platform-comparison.md)
- reset/migration：  
  [GPU VM 生命周期、热插拔、Checkpoint 与迁移](vm-lifecycle-migration.md)
