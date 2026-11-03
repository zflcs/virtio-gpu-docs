# OpenCL、Level Zero、SYCL 与 AI compute 虚拟化边界

## 1. 文档目标

AI compute 不只等于 CUDA 和 ROCm。实际部署还会出现：

- OpenCL；
- Intel oneAPI/Level Zero；
- SYCL；
- Vulkan compute；
- 以上 API 之上的 oneDNN、oneMKL、框架 XPU backend 或自定义 kernel。

本文回答：

- 这些名称分别属于应用 API、编译器、runtime、驱动还是产品集合；
- 虚拟化一个运行 CUDA/HIP 的程序，与虚拟化一个 OpenCL/SYCL 程序有何不同；
- 为什么 SYCL “支持某 backend”不等于现成二进制自动兼容；
- 在 virtio-gpu 的 VirGL、Venus、GFXStream、native DRM/ROCm 路径上，哪些
  API 可以直接工作，哪些必须翻译或重新编译；
- 如何用源码、ABI、extension 和运行证据评审 AI compute 兼容性。

固定源码：

```text
Mesa ec7b2f322abbcd593d600a14afe2d2773512af4e
Linux 6b4b543fa18757297ac756a6fe6e64f506b7844a
```

本地可读源码证据主要覆盖 Mesa Rusticl/OpenCL 和 Linux Xe UAPI。
Level Zero、SYCL 和 oneAPI 的精确版本支持必须再绑定具体 SDK、driver 和
产品文档；本文不虚构其私有 implementation。

本文不运行 OpenCL kernel、SYCL benchmark、GPU VM 或编译实验。

## 2. 先把名称放回正确层次

| 名称 | 主要层次 | 是否单一 API | 关键契约 |
| --- | --- | --- | --- |
| CUDA | runtime + driver API + vendor ecosystem | 是 | CUDA ABI、driver ABI、cubin/PTX |
| ROCm | runtime + libraries + KFD/ROCr/libhsakmt | 产品/栈 | HIP/ROCr ABI、KFD UAPI |
| OpenCL | 跨厂商 C API + ICD | 是 | ICD ABI、kernel/object 语义 |
| Vulkan compute | graphics/compute API | 是 | Vulkan ABI、SPIR-V、sync/layout |
| Level Zero | Intel low-level compute API | 是 | Level Zero C ABI、device/context/queue |
| oneAPI | 工具、库、runtime 和产品集合 | 否 | 取决于具体组件 |
| SYCL | C++ 编程模型/标准 | 否 | 源码编译、backend、device image |

最重要的纠错：

```text
oneAPI != Level Zero
SYCL != 一个固定 runtime
OpenCL ICD != CUDA driver ABI
Vulkan compute != CUDA/HIP compatibility
```

oneAPI 可能包含 Level Zero、OpenCL、SYCL、MKL/DNN、CCL、compiler 和 profiler。
判断兼容性必须继续问“具体哪个 library/runtime/API”。

## 3. 五种兼容性

### 3.1 Source/API compatibility

源代码重新编译后可用：

```text
SYCL source
  -> chosen SYCL implementation/compiler
  -> backend selection
  -> OpenCL, Level Zero, CUDA, HIP, Vulkan or other backend
  -> device code and runtime objects
```

SYCL 的最大价值是源码层可移植，不是把所有 backend 变成相同 ABI。

### 3.2 Binary ABI compatibility

已有 `.so`/executable 可加载：

- OpenCL ICD dispatch 满足 loader 期望；
- Level Zero loader/driver ABI 匹配；
- SYCL runtime ABI 匹配；
- HIP/CUDA runtime/driver ABI 匹配；
- shared library soname/version 匹配。

只有源码兼容时，应重新编译；不能把旧二进制直接迁移。

### 3.3 Intermediate/kernel compatibility

不同栈的中间表示：

| 栈 | 常见中间/二进制 |
| --- | --- |
| CUDA | PTX、cubin/SASS |
| HIP/AMD | LLVM IR、AMDGPU code object |
| OpenCL | implementation binary、SPIR-V |
| Vulkan | SPIR-V + pipeline/descriptor state |
| Level Zero | module/native binary + kernel objects |
| SYCL | implementation-defined device image/offload bundle |

中间表示兼容仍不保证 extension、ABI 或 runtime object 兼容。

### 3.4 Driver/UAPI compatibility

以下对象必须由 Guest/Host 双方正确理解：

- device discovery；
- context/process；
- command queue/list；
- memory allocation/import；
- event/timeline/fence；
- kernel/module；
- profiling/debug；
- reset/error。

### 3.5 Hardware/ISA compatibility

编译后的 device code 是否能在目标 GPU 执行，取决于 architecture、
firmware、driver compiler 和 runtime。Virtio 设备能力转发成功不能替代
ISA/profile 检查。

## 4. API 对象模型对照

| 概念 | CUDA | ROCm | OpenCL | Level Zero | Vulkan compute |
| --- | --- | --- | --- | --- | --- |
| Device | CUDA device | KFD node/GPU | CL device | ze_device | VkPhysicalDevice |
| Context | CUcontext | KFD process/VM | cl_context | ze_context | VkDevice/state |
| Queue | stream | AQL/SDMA queue | cl_command_queue | command queue/list | VkQueue |
| Work | kernel launch | AQL dispatch | NDRange kernel | kernel launch | dispatch command |
| Kernel | module/function | code object | program/kernel | module/kernel | pipeline/shader |
| Memory | device/pinned/UM | VRAM/GTT/SVM | buffer/image/SVM | USM/device memory | buffer/image/device memory |
| Completion | event/fence | event/fence | cl_event | event/fence | fence/timeline |
| Interop | CUDA IPC/VMM | HSA/dma-buf | GL/dma-buf ext | IPC/import extensions | external memory/semaphore |

对象名称相近不代表 ABI 相同。虚拟化方案必须明确转发的是哪一层。

## 5. Rusticl/OpenCL 源码路径

Mesa Rusticl 是一个开源 OpenCL frontend，适合观察“标准 compute API 如何落到
Gallium driver/KMD”。

### 5.1 ICD dispatch

[`cl_dispatch!`](../../mesa/src/gallium/frontends/rusticl/api/icd.rs:82)
建立标准 OpenCL entrypoint 表，包括：

- `clCreateContext`
- `clCreateCommandQueue`
- `clCreateBuffer`
- `clCreateProgramWithSource`
- `clCreateKernel`
- `clEnqueueNDRangeKernel`
- event、flush、finish、read/write、GL interop。

Rusticl 构建为 OpenCL ICD library；系统 loader 如何选择它属于 loader 配置层。

### 5.2 Context 和 queue

[`clCreateContext`](../../mesa/src/gallium/frontends/rusticl/api/context.rs:115)
验证：

- device 列表；
- platform、interop 和 EGL/GLX properties；
- context 与 device 的关联。

[`clCreateCommandQueue`](../../mesa/src/gallium/frontends/rusticl/api/queue.rs:119)
校验 queue properties 和 device 能力。OpenCL 队列可以是 in-order/out-of-order
或 profiling；这些语义不能只靠 Host 上“能提交 kernel”证明。

### 5.3 Memory

[`clCreateBuffer`](../../mesa/src/gallium/frontends/rusticl/api/memory.rs:412)
处理：

- allocation flags；
- size；
- host pointer；
- sub-buffer；
- device-address extension。

OpenCL 还支持 image、map/unmap、read/write、copy、SVM、external memory/interop。
虚拟化必须分别验证，不只是 buffer allocation。

### 5.4 Program 和 compile

[`clCreateProgramWithSource`](../../mesa/src/gallium/frontends/rusticl/api/program.rs:131)
保存 OpenCL C source。
[`clBuildProgram`](../../mesa/src/gallium/frontends/rusticl/api/program.rs:309)
触发 device-specific build。

固定 Rusticl 源码的编译路径：

```text
OpenCL C source
  -> Mesa CLC compilation
  -> SPIR-V
  -> SPIR-V-to-NIR
  -> Gallium driver shader compilation
  -> GPU code
```

关键函数：

- [`SPIRVBin::from_clc()`](../../mesa/src/gallium/frontends/rusticl/mesa/compiler/clc/spirv.rs:106)
- [`SPIRVBin::to_nir()`](../../mesa/src/gallium/frontends/rusticl/mesa/compiler/clc/spirv.rs:333)
- [`Program::to_nir()`](../../mesa/src/gallium/frontends/rusticl/core/program.rs:242)
- [`Program::do_compile()`](../../mesa/src/gallium/frontends/rusticl/core/program.rs:780)

Rusticl 也接受 SPIR-V program IL。接收 SPIR-V 不等于可执行：还必须通过
version/capability/extension 校验并进入后端 shader compiler。

### 5.5 Kernel launch

[`clEnqueueNDRangeKernel`](../../mesa/src/gallium/frontends/rusticl/api/kernel.rs:697)
验证：

- queue/kernel context 一致；
- program 已成功 build；
- kernel 参数齐全；
- work dimension；
- local/global size 和 work-group 限制；
- event wait list；
- kernel 引用在异步执行期间保活。

[`Kernel::launch_with_args()`](../../mesa/src/gallium/frontends/rusticl/core/kernel.rs:1685)
冻结参数、buffer/image 引用和 NIR variant。大 grid 会按硬件最大 grid 切成
多个 dispatch，并更新 workgroup ID offset。

最终调用：

```text
Kernel launch
  -> ctx.launch_grid()
  -> pipe_context::launch_grid()
  -> Gallium driver/KMD
  -> GPU compute queue
```

[`PipeContext::launch_grid()`](../../mesa/src/gallium/frontends/rusticl/mesa/pipe/context.rs:504)
把 block、grid、shared memory 和 global bindings 封装为 `pipe_grid_info`。

## 6. Intel Xe UAPI 对照

当 Rusticl 使用的 Gallium driver 最终落到 Intel Xe KMD 时，底层不是
OpenCL ioctl，而是 DRM Xe UAPI。

[`struct drm_xe_gem_create`](../../linux/include/uapi/drm/xe_drm.h:843)
创建 GEM object，可声明 VRAM/system memory placement、PXP 等属性。

[`struct drm_xe_vm_bind`](../../linux/include/uapi/drm/xe_drm.h:1230)
执行：

- MAP；
- MAP_USERPTR；
- UNMAP；
- PREFETCH；
- 只读、immediate fault、CPU address mirror 等属性。

[`struct drm_xe_exec_queue_create`](../../linux/include/uapi/drm/xe_drm.h:1433)
创建 execution queue，并可设置 priority、timeslice、PXP 和 multi-queue group。

[`struct drm_xe_exec`](../../linux/include/uapi/drm/xe_drm.h:1623)
提交 batch buffer，并用 syncobj/timeline syncobj/user fence 表达依赖和完成。

源码调用链示例：

```text
OpenCL or Level Zero frontend
  -> userspace driver/compiler
  -> GEM object and VM bind
  -> exec queue
  -> drm_xe_exec
  -> GuC/GPU
```

这条链保留的是 OpenCL/L0 用户态 API + Xe UAPI 组合。把 Guest 的 Xe UAPI
换成 Virtio-GPU 虚拟设备后，原用户态 stack 是否继续工作取决于是否有相应
proxy/translation。

## 7. Level Zero 是什么

Level Zero 是 Intel 的低层 compute API，公开对象通常包括：

- driver/device discovery；
- context；
- command queue 和 command list；
- event/fence；
- module/kernel；
- device/shared/host USM memory；
- IPC handle；
- memory migration/advice；
- profiling/tracing。

典型 API 概念路径：

```text
zeInit / zeDriverGet
  -> zeDeviceGet
  -> zeContextCreate
  -> zeCommandQueueCreate
  -> zeCommandListCreate
  -> zeModuleCreate / zeKernelCreate
  -> zeCommandListAppendLaunchKernel
  -> queue execute + event/fence
```

Level Zero 的价值是暴露比 OpenCL/SYCL 更直接的 compute/queue/USM 控制。

### 7.1 虚拟化 Level Zero 的难点

如果 Guest 原生 stack 需要使用 Level Zero，必须至少保留：

- loader/driver discovery；
- device capability 和 UUID；
- context/queue/list；
- USM allocation/map/advice；
- module/kernel binary；
- event/fence；
- IPC/import/export；
- error/reset。

当前本文覆盖的 virtio-gpu VirGL、Venus、GFXStream、native DRM 和 ROCm
proposal 中，没有一条已描述的 capset 直接代理 Level Zero ABI。

因此 Level Zero workload 的基本选择是：

1. VFIO/VF/vGPU/MIG 受支持组合，让 Guest 使用原生 Intel stack；
2. Guest 重新选择另一种可用 backend；
3. 新设计一个 Level Zero proxy，并完整实现对象、内存和同步协议。

不能把“Vulkan compute 可用”或“OpenCL 能跑一个 kernel”写成“Level Zero
兼容”。

## 8. SYCL 是什么，不是什么

SYCL 是建立在 C++ 异步执行、buffer/accessor/USM、queue/event 等概念上的
编程模型。SYCL implementation 负责：

- host/device code 编译；
- device image/bundle；
- backend plugin 选择；
- memory 和 event 映射；
- kernel launch。

一个 implementation 可能使用 OpenCL、Level Zero、CUDA、HIP 或其他 backend。
因此：

```text
SYCL program
  -> implementation/version
  -> backend plugin
  -> backend runtime + KMD
```

### 8.1 Source portability

修改少量 backend-specific 代码后重新编译，已经很有价值。

### 8.2 Binary portability

已有 SYCL 可执行文件通常包含实现特定 runtime/device image。换 GPU 或换
virtualization backend 后，不一定能直接执行。

### 8.3 oneAPI library 依赖

oneDNN、oneMKL、oneCCL 等库可能：

- 有独立 runtime 初始化；
- 使用 Level Zero/OpenCL/SYCL backend；
- 依赖 device capability；
- 有自己的 kernel cache；
- 做 topology/collective 查询。

即使应用用 SYCL 源码写成，第三方 oneAPI library 的 compatibility 仍要
逐库确认。

## 9. Vulkan compute 与 OpenCL/SYCL

Vulkan compute 对象模型：

- device/queue；
- descriptor set/layout；
- pipeline layout/pipeline；
- command buffer；
- buffer/image；
- semaphore/fence/timeline；
- external memory。

Venus 和 GFXStream 主要代理 Vulkan API/wire command。它们可以承载一部分：

- 自己编写并转为 SPIR-V 的 compute kernel；
- 标准 Vulkan buffer/image；
- 不依赖 CUDA/HIP/Level Zero 的 AI kernel。

不能自动提供：

- OpenCL ICD ABI；
- Level Zero ABI；
- CUDA/HIP runtime；
- oneAPI library 内部私有 API；
- NCCL/RCCL collective。

## 10. 三种“OpenCL 跑在 virtio-gpu 上”的方案

### 10.1 Guest 内 Rusticl + 可用 Gallium driver

```text
OpenCL app
  -> ICD loader
  -> Rusticl
  -> Guest Gallium driver
  -> Guest KMD
```

如果 Guest Gallium driver/KMD 是 virtio-gpu，compute 能力取决于它是否实现
所需 Gallium compute/launch_grid paths。不能仅因 Rusticl 源码存在就断定
VirGL capset 支持完整 OpenCL。

### 10.2 CLVK 翻译到 Vulkan/Venus

```text
OpenCL app
  -> OpenCL ICD
  -> CLVK
  -> Vulkan API
  -> Venus/GFXStream or native Vulkan
```

这是 API translation，不是 OpenCL remoting：

- OpenCL object 被映射到 Vulkan object；
- kernel 需要转为 Vulkan compute SPIR-V；
- memory flag、image、event 和 extension 存在语义差异；
- 只支持 CLVK 能映射的功能子集。

如果 Host Venus 的 Vulkan capability 足够，应用源码可能无需修改，但这需要
逐功能验证，不是 ABI 保证。

### 10.3 Vendor OpenCL runtime + passthrough VF

```text
OpenCL app
  -> vendor ICD
  -> Guest vendor UMD
  -> Guest vendor KMD
  -> VFIO physical GPU/VF
```

这条路径最容易保留 vendor OpenCL binary/extension，但牺牲多租密度，并依赖
Guest driver/firmware 支持。

## 11. AI workload 到 API/backend 的判断

### 11.1 PyTorch/JAX/TensorFlow

框架不是一个 API。它可能通过：

- CUDA backend；
- ROCm/HIP backend；
- oneAPI/Level Zero/XPU backend；
- CPU；
- custom extension；
- collective library。

“模型能加载”不代表所选 backend 的 kernel/library 已兼容。

### 11.2 Intel AI stack

部署 Intel GPU 时，版本矩阵通常同时包含：

- framework XPU backend；
- oneDNN/oneMKL/oneCCL；
- SYCL implementation；
- Level Zero/OpenCL runtime；
- userspace driver；
- Xe/i915 KMD 和 firmware。

替换其中一个组件不能只检查另一个组件版本号。

### 11.3 Reference kernels

一个手写 OpenCL/SYCL/Vulkan kernel 的迁移难度与完整框架不同：

| Workload | 更容易的路径 | 主要额外问题 |
| --- | --- | --- |
| 自研 OpenCL kernel | Rusticl/CLVK/vendor OpenCL | SPIR-V/extension、内存、queue |
| 自研 SYCL kernel | 同 backend 重新编译 | implementation/device image |
| 自研 Vulkan compute | Venus/GFXStream | descriptor/layout、timeline |
| 自研 Level Zero | 原生 L0 + native GPU | 无现成 virtio proxy |
| PyTorch CUDA | VFIO/VF/vGPU | 完整 CUDA/NCCL |
| PyTorch Intel XPU | 原生 Intel stack | L0/SYCL/oneAPI 版本 |
| vLLM 训练/推理 | 完整厂商 runtime | collective、graph、KV cache |

## 12. 虚拟化机制横向矩阵

| 机制 | OpenCL ABI | Level Zero ABI | SYCL source | Vulkan compute | CUDA/HIP |
| --- | --- | --- | --- | --- | --- |
| VFIO 整卡 | Guest vendor ICD 可保留 | Guest Intel stack 可保留 | 可重新编译/使用 backend | Guest Vulkan | 完整栈 |
| SR-IOV VF | 依 VF profile | 依 Intel VF stack | 依 backend | Guest Vulkan | Vendor VF stack |
| vGPU/MIG | 依产品 | 依产品 | 依产品 | 依产品 | 依产品 |
| VirGL | 无标准 OpenCL capset | 无 | 无直接保证 | 不是 Vulkan remoting | 无 |
| Venus | 无直接 OpenCL；可评估 CLVK | 无 | 仅当 backend 可用 | 原生目标 | 无 |
| GFXStream | 无标准 OpenCL proxy | 无 | 依 backend | Vulkan streaming | 无 |
| native DRM | Guest Mesa/vendor ICD 依 UAPI | 依 frontend | 依 backend | 可配 Vulkan | 不等同 CUDA/HIP |
| ROCm proposal | 无 | 无 | 无 | 无 | Guest ROCm/HIP 方向 |

“可保留”仍需要具体 Guest/Host driver、device profile 和版本矩阵。

## 13. 虚拟化 API 验收矩阵

### 13.1 Discovery

- application 找到预期 loader/ICD/backend；
- device count、name、UUID、topology 正确；
- API version、extension、subgroup、USM/SVM 能力正确；
- 没有错误回退到 CPU。

### 13.2 Object lifecycle

- context、queue、program/module、kernel 创建/销毁；
- queue 并发和 ordering；
- event/fence wait、reset、profiling；
- process teardown 释放 Host/GPU 对象。

### 13.3 Compilation

- source、SPIR-V、native binary、cache；
- driver compiler 版本；
- device architecture；
- extension 和 specialization constant；
- 编译失败日志是否可传播。

### 13.4 Memory

- device/local memory；
- host-visible/pinned；
- USM/SVM/shared virtual address；
- map/read/write/copy；
- external memory/dma-buf/IPC；
- migration、eviction、fault、TLB invalidation。

### 13.5 Execution

- NDRange/global-local work size；
- dispatch、barrier、atomic；
- event dependency；
- concurrent queue；
- long kernel、preemption、reset。

### 13.6 AI stack

- framework device selection；
- oneDNN/oneMKL kernel 路径；
- oneCCL/NCCL/RCCL；
- graph/JIT/cache；
- inference/训练；
- checkpoint/restart。

## 14. 失败定位

| 症状 | 优先层 |
| --- | --- |
| Device 不可见 | ICD/loader、backend plugin、DRM node、PCI/VF |
| 找到 device 但 context 失败 | API 与 Guest driver ABI |
| program build 失败 | compiler、SPIR-V、extension、ISA |
| kernel 参数/launch 失败 | work size、descriptor、queue property |
| 结果错误但无报错 | memory alias、cache、barrier、layout |
| event 永不完成 | queue/doorbell、fence 映射、Host callback |
| 回退 CPU | backend capability 或 plugin 选择 |
| OpenCL 在 Venus 失败 | CLVK 子集、Vulkan feature、SPIR-V |
| Level Zero 不可用 | 当前没有相应 virtio proxy 或原生 driver |
| SYCL binary 失败 | implementation/backend/device image 不匹配 |
| oneAPI library 失败 | 版本矩阵、独立 backend/collective |
| 单 kernel 正常、模型异常 | graph、library、allocator、collective |

## 15. 常见错误结论

1. “OpenCL 是跨厂商 API，所以 virtio-gpu 自动支持 OpenCL。”
2. “Rusticl 是 Mesa frontend，所以任何 Mesa GPU 都有完整 OpenCL。”
3. “CLVK 能跑一个 kernel，所以整个框架 OpenCL 路径可用。”
4. “Vulkan compute 可用，所以 CUDA/HIP/Level Zero 二进制能运行。”
5. “SYCL 是标准，所以 SYCL binary 能在任意 backend 运行。”
6. “oneAPI 是 API，所以只需代理一个 ABI。”
7. “Level Zero 在 Xe 上使用 DRM UAPI，所以 virtio native DRM 自动支持 L0。”
8. “CUDA Graph/PyTorch custom extension 可以像 SPIR-V kernel 一样翻译。”
9. “单卡 GEMM 性能正常，所以 oneCCL/NCCL/RCCL 正常。”
10. “device name 显示 Intel/NVIDIA/AMD，所以所选 loader 和 backend 一定正确。”

## 16. 检查题

1. OpenCL ICD、Level Zero loader 和 CUDA driver ABI 分别保留什么？
2. 为什么 SYCL 源码可移植不等于 binary ABI 可移植？
3. oneAPI 为什么不能作为单一虚拟化 API 处理？
4. Rusticl 如何把 OpenCL C 转到 SPIR-V/NIR？
5. `pipe_context::launch_grid()` 在源码路径中代表什么边界？
6. `drm_xe_vm_bind`、exec queue 和 `drm_xe_exec` 分别是什么对象？
7. Level Zero USM/IPC 对 virtio-gpu memory proxy 提出什么要求？
8. CLVK 与 OpenCL API remoting 的本质区别是什么？
9. 为什么 Venus 能执行 Vulkan compute 却不一定能运行 OpenCL 应用？
10. VFIO 整卡为什么最容易保留 vendor OpenCL/Level Zero binary？
11. oneAPI library 为什么可能突破底层 SYCL backend 的兼容假设？
12. 如何区分 program build 失败和 driver launch 失败？
13. 哪些证据能证明 external memory/dma-buf 真的可用？
14. 如何验证推理 decode 与训练 collective 的不同瓶颈？
15. 设计一个从 OpenCL API 到 GPU queue 的虚拟化验收矩阵。

## 17. 与其他专题

- [AI Runtime、API 与虚拟化兼容性](ai-runtime-api-compatibility.md)
- [AI GPU 计算栈与虚拟化边界](ai-compute-virtualization.md)
- [GPU 执行模型、队列、调度与故障隔离](gpu-execution-model.md)
- [GPU 显存、地址空间与一致性语义](gpu-memory-semantics.md)
- [Intel Xe GPU 虚拟化与 SR-IOV](intel-xe-sriov.md)
- [Venus 源码跟踪](../virtio-gpu/venus-source-trace.md)
- [GFXStream 源码跟踪](../virtio-gpu/gfxstream-source-trace.md)
- [Virtio-GPU ROCm compute 源码跟踪](../virtio-gpu/rocm-source-trace.md)
