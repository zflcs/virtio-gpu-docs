# GPU 显存、地址空间与一致性语义

## 1. 文档目标与源码基线

本文把 virtio-gpu、AMD ROCm/KFD、VFIO、SR-IOV、MIG 和厂商 vGPU 放在同一套
内存语义下比较。目标不是记住某条 ioctl，而是遇到任何 GPU workload 时都能
回答：

1. 这个地址是 Guest CPU VA、GPA、Host CPU VA、DMA/IOVA、GPU VA，还是
   VRAM 内部的物理地址？
2. 数据当前驻留在系统内存还是 GPU 显存？
3. 谁建立了 CPU、IOMMU 和 GPU 页表中的映射？
4. CPU cache 与 GPU cache 在何时保持一致？
5. 哪个 fence 证明数据已被 GPU 使用完，哪个 fence 只证明命令已被 Host
   消费？
6. CPU unmap、进程退出、GPU eviction 或设备 reset 时，哪一方必须先停止
   访问？

源码快照：

- Linux `6b4b543fa18757297ac756a6fe6e64f506b7844a`
- Mesa `ec7b2f322abbcd593d600a14afe2d2773512af4e`
- virglrenderer `e0a1caf3b147bb26dfe524dbb85ded1177e7e49e`
- QEMU `d73149283205645625bfa9fe029cfb68e4c4a9ae`

本文只读取源码和文档，不运行 VM、构建、benchmark 或故障实验。

## 2. 一张图理解多个地址域

Guest 中的一次 GPU 访问通常不是“一个地址从应用一路传到 GPU”，而是多张
页表和多个地址域依次转换：

```text
Guest CPU VA
    |
    | Guest CPU page table
    v
Guest physical address (GPA)
    |
    | virtio descriptor / mem_entry
    | QEMU device DMA address space
    v
Host CPU VA used by QEMU/renderer
    |
    | optional DMA API / IOMMU
    v
DMA address / IOVA
    |
    | IOMMU page table
    v
Host physical address (HPA)
```

GPU 执行 work 时还有另一条常见路径：

```text
GPU VA in a kernel/buffer descriptor
    |
    | GPU page table / GPU MMU
    +--> VRAM physical location
    |
    +--> system-memory address
             |
             | IOMMU / interconnect
             v
          Host memory
```

具体实现可能把某些阶段合并或省略，但不能假设这些地址相等。

### 2.1 术语表

| 名称 | 谁使用 | 作用 |
| --- | --- | --- |
| Guest CPU VA | Guest 进程 | CPU load/store、HIP/CUDA/Vulkan API 的用户地址 |
| GPA | Guest 设备模型 | Guest 的物理地址空间，QEMU 用内存区域表示 |
| Host CPU VA | QEMU、renderer、Host runtime | Host 进程访问 guest RAM 或 Host 分配的缓冲区 |
| DMA address / IOVA | 需要 DMA 的设备 | 设备提交给总线/IOMMU 的地址 |
| HPA | Host 内存管理 | Host 系统内存的物理地址 |
| GPU VA | shader、compute kernel、GPU 页表 | GPU 内部虚拟地址 |
| VRAM/HBM address | GPU memory controller | 显存内部地址或 allocation offset |
| GTT/aperture | GPU 页表 | GPU 访问系统内存的窗口/映射区域 |
| ZONE_DEVICE page | Linux MM | 用 `struct page` 表示 device-private memory，参与迁移和 fault |

### 2.2 为什么 virtio-gpu 的 mem entry 不一定是 GPA

[`virtio_gpu_object_shmem_init()`](../../linux/drivers/gpu/drm/virtio/virtgpu_object.c:171)
在 `virtio_gpu_use_dma_api()` 为真时，把 `sg_dma_address()` 写入
`virtio_gpu_mem_entry.addr`；否则写 `sg_phys()`。

这说明 mem entry 的数值取决于 Guest 平台的 DMA API 使用方式，而不应仅凭
字段名把它当作固定类型的地址。QEMU 随后在
[`virtio_gpu_create_mapping_iov()`](../../qemu/hw/display/virtio-gpu.c:983)
中通过 `dma_memory_map()` 和设备的 DMA address space 解释这些地址，得到
QEMU/renderer 可使用的 Host VA 或映射描述。

地址翻译成功只说明“怎样访问这个地址”，不说明页面仍然驻留，也不说明 cache
一致。

## 3. 五个必须分开的维度

审查任何 GPU 内存路径时，分别判断下面五项。把它们合成一个
“zero-copy=true”或“memory=true”之类的布尔量通常会导致错误结论。

### 3.1 Allocation：谁拥有对象

典型对象包括：

- Guest GEM/DRM object、Vulkan `VkDeviceMemory`、HIP allocation；
- QEMU resource、virglrenderer resource、Host Vulkan memory；
- Host GPU BO、dma-buf、userptr registration。

一个 Guest 对象和一个 Host 对象可以对应同一个 backing，也可以是两个独立
allocation。例如：

- `VIRTGPU_BLOB_MEM_GUEST` 有 Guest backing；
- `VIRTGPU_BLOB_MEM_HOST3D` 有 Host renderer backing；
- `VIRTGPU_BLOB_MEM_HOST3D_GUEST` 同时有两侧描述。

一侧有句柄不代表另一侧拥有同一片物理页。

### 3.2 Mapping：地址如何翻译

映射至少可能包括：

- CPU 页表把 Guest CPU VA 映射到 GPA；
- VMCS/EPT 或 stage-2 把 GPA 映射到 HPA；
- IOMMU 把设备 IOVA 映射到 HPA；
- GPU MMU 把 GPU VA 映射到 VRAM 或 system-memory address；
- Host CPU 页表把 Host VA 映射到 HPA 或映射设备 BAR。

映射不是 residency。一个 GPU VA 可以有合法 PTE，但 PTE 指向 system memory；
反过来，数据可以在 VRAM 中存在但暂时没有可访问的 GPU VA。

### 3.3 Residency：数据在哪里

常见位置：

- Guest RAM；
- Host anonymous/file-backed memory；
- Host pinned memory；
- device-private VRAM/HBM；
- peer GPU memory；
- migration staging、bounce buffer 或 swap。

SVM/HMM 可以把同一个 CPU VA 的 backing 从 system memory 迁移到 device
memory，再迁回系统内存。迁移改变物理 residency，不应改变 API 看到的 VA。

### 3.4 Coherence：谁看见什么

coherence 至少涉及：

- CPU cache 与 memory controller；
- GPU L1/L2 与 HBM/GTT；
- PCIe/CXL/peer fabric 上的 ordering；
- IOMMU 是否存在；
- DMA API 是否需要 cache maintenance 或 SWIOTLB；
- 图形 API 在 barrier/fence 处定义的可见性。

DMA 映射、PTE 中的 snoop 属性、API barrier 和 `dma_fence` 解决的是不同层面
的问题，不能互相替代。

### 3.5 Lifetime：谁保证仍未释放

至少同时考虑：

- GEM/resource 引用；
- Host renderer 对象引用；
- page pin 或 migration reference；
- MMU notifier registration；
- pending command 对 backing 的引用；
- fence completion 和 teardown 顺序。

“map 成功”不自动持有足够长的生命周期保证。长 compute job 尤其可能超出
普通 frame resource 的假设。

## 4. Virtio-GPU 的几类 backing

### 4.1 普通 shmem resource

当前主线的普通 GEM 对象由
[`virtio_gpu_object_create()`](../../linux/drivers/gpu/drm/virtio/virtgpu_object.c:213)
创建，并通过 attach backing 把 mem entry 发给设备。

概念路径：

```text
Guest drm_gem_shmem pages
  -> mem_entry[]
  -> virtio control queue
  -> QEMU dma_memory_map
  -> renderer resource backing
```

但它不是所有图形操作都自动共享同一 GPU PTE。VirGL 可通过 transfer 命令
上传/回读 texture/buffer；Venus/GFXStream 还可能有自己的 Host allocation、
blob 和 dma-buf 路径。

### 4.2 `VIRTGPU_BLOB_MEM_GUEST`

语义重点是 backing 来自 Guest。Guest blob 可以伴随 mem entry，Host
renderer 能看到 Guest storage 的映射描述。

这并不意味着 Host GPU 会无条件直接访问 Guest 页：

- driver 可能把数据复制到 Host-created resource；
- QEMU/DMA 层可能产生 bounce；
- Host renderer 可能选择 system-memory GTT 而不是 VRAM；
- Guest 页可以被 pin 或仍可迁移，取决于具体路径。

### 4.3 `VIRTGPU_BLOB_MEM_HOST3D`

Host renderer/context 创建 Host-side storage。Guest 通过 resource ID 引用它；
需要 CPU 可见时，通常再使用 mappable hostmem 窗口。

Venus 的 ring 和 GFXStream 的 ASG ring 常用这种 mappable blob：

```text
Host-created mappable blob
  -> QEMU hostmem MemoryRegion
  -> Guest mmap
  -> Guest 写 ring 内容
  -> Guest 发布 tail
```

这里的共享地址主要是控制/命令 ring，不是训练张量本身的通用显存分配器。

### 4.4 `VIRTGPU_BLOB_MEM_HOST3D_GUEST`

两侧都有 backing 描述。Host renderer 可能在 Host 创建 device resource，同时
使用 Guest iovec 或 dma-buf 作为共享 backing。真正是否零拷贝，要沿具体
backend、fd 类型和 GPU PTE 检查。

### 4.5 Blob flags

当前 UAPI 中可见：

- `USE_MAPPABLE`：允许 Host 返回可在 Guest 映射的窗口；
- `USE_SHAREABLE`：允许 dma-buf 类跨设备/进程共享；
- `USE_CROSS_DEVICE`：面向 cross-domain 资源；
- `BLOB_FLAG_HINT_DEFER_MAPPING`：延迟映射，而不是立即建立所有物理映射。

这些 flag 描述能力和意图，不证明某一数据的 DMA coherence 或 residency。

### 4.6 dma-buf

dma-buf 是跨驱动/进程共享 backing 的 fd 抽象。它解决对象共享和 import/export
契约，仍需要：

- export/import 两侧格式和 modifier 兼容；
- DMA 映射方向正确；
- 访问期间 object/lifetime 仍有效；
- acquire/release 或 API barrier 同步；
- 各 GPU 页表建立正确 PTE。

fd 可传递不等于所有设备和 CPU 永远看到同一份最新 cache line。

### 4.7 拟议的 virtio-gpu userptr

截至 2026-10-29，Linux virtio-gpu userptr RFC v9 尚未合入主线，讨论不能当作
当前运行语义。其设计方向是对 Guest 用户 VA 范围做 long-term pin，并生成
Guest blob backing：

```text
Guest user VA range
  -> pin_user_pages / SG table
  -> virtio userptr blob
  -> QEMU/renderer/Host userptr 或 dma-buf path
  -> Host GPU PTE
```

该方向减少了“先在 Guest 分配一份 shmem staging 再复制”的一次分配和 memcpy，
但没有自动保证：

- QEMU 不建立额外映射；
- DMA API 不使用 bounce buffer；
- 数据驻留在 VRAM；
- GPU 与 CPU 连续 coherent；
- 用户可以在 resource 存活时任选 munmap、mremap 或迁移；
- Host 与 Guest 页大小/alignment 天然兼容。

RFC 评审中关于 `DMA_ATTR_REQUIRE_COHERENT` 的讨论，正是因为连续 user-memory
DMA coherence 并非所有平台都能仅靠页表属性实现。详见
[Virtio-GPU Userptr 与 ROCm 社区动向](../virtio-gpu/userptr-rocm-compute.md)。

## 5. 四条图形路径的内存差异

### 5.1 VirGL

VirGL 的资源通常有 Guest GEM backing，Host vrend 建立 host GL resource。
数据可能通过：

- `RESOURCE_ATTACH_BACKING` 让 Host 获得 Guest mem entry；
- `TRANSFER_TO_HOST_3D` 上传；
- `TRANSFER_FROM_HOST_3D` 回读；
- mappable blob 和 hostmem window 直接映射命令/共享数据。

关键语义：

- Guest 发 transfer 后要等对应 fence 才能认为 Host 侧操作完成；
- Host GL texture 已经存在不代表 Guest CPU mapping 中的数据已经可见；
- attach backing、resource 引用和 pending GL work 必须覆盖释放周期。

### 5.2 Venus

Venus 的主要命令流位于共享 ring blob：

```text
Guest writes command bytes
  -> release-store tail
  -> notify Host
  -> Host acquire-load tail, execute dispatch
  -> release-store head
```

ring head 只说明 Host 已消费 command。Vulkan device memory 仍是另一套对象：

- 普通 allocation 可由 Host Vulkan 创建；
- Guest/Host blob、dma-buf 或 opaque fd 可被 import；
- `vkr_device_memory` 根据 memory type 和 external capability 设置
  cached/coherent 属性；
- fence/semaphore/barrier 决定可见性。

因此“ring 已映射”不能推出“所有 Vulkan memory 都由同一块 Guest 页直接
backing”。

### 5.3 GFXStream

GFXStream 的 ASG blob 用于 API command bytes，ResourceTracker 另外管理 image、
buffer、device memory 和 external memory。

审查时必须分开：

1. ASG ring 的 Guest/Host 共享映射；
2. Guest Vulkan object ID 到 Host boxed handle；
3. Host image 的 memory requirements、tiling 和 modifier；
4. compositor/WSI 使用的 dma-buf；
5. queue submit fence。

ring 通路工作、image allocation 成功和 native image 可采样是三个不同结论。

### 5.4 Rutabaga

Rutabaga 是 QEMU 侧的 VGI/component 抽象，不应把“Rutabaga resource”当作另一种
GPU 显存硬件。它把 blob、map 和 fence 请求转给选中的 component；实际 backing
仍由 component 或 native renderer 决定。

### 5.5 横向表

| 路径 | 命令/控制内存 | 图像或计算数据 | 典型同步 |
| --- | --- | --- | --- |
| VirGL | command payload + attach backing | Guest GEM、transfer、blob | virtio fence、GL completion |
| Venus | mappable ring blob | Host Vulkan memory、blob/dma-buf | ring seqno + Vk fence/semaphore |
| GFXStream | ASG ring blob | ResourceTracker、Host Vulkan/external memory | GFXStream seqno + runtime fence |
| Rutabaga | QEMU command + component blob | 由 component 决定 | component fence callback |

## 6. ROCm/KFD：从 UAPI 到 GPU 页表

### 6.1 ROCm 内存类型

[`kfd_ioctl.h`](../../linux/include/uapi/linux/kfd_ioctl.h) 用不同 flag 表达：

- VRAM；
- GTT；
- USERPTR；
- doorbell/MMIO remap；
- writable/executable/public；
- coherent、uncached、extended coherent；
- contiguous。

这些 flag 与 SVM 属性是不同层次。`KFD_IOC_ALLOC_MEM_FLAGS_USERPTR` 表示
allocation 如何取得 backing；`KFD_IOCTL_SVM_FLAG_COHERENT` 表达跨设备的
细粒度一致性要求；PTE 的 snoop/MTYPE 是硬件实现属性。

### 6.2 SVM 不只是一个 “malloc”

KFD SVM 把一个 CPU VA range 变成可由 GPU 访问的 range，并维护：

- CPU VMA 与 GPU range/notifier；
- GPU VA 到 system memory 或 VRAM 的页表；
- preferred/prefetch/access policy；
- migration 和 eviction；
- GPU retry fault；
- queue quiesce/restore；
- 与 CPU mmap/munmap 的竞态安全。

GPU 可能和 CPU 使用同一个 VA，但背后仍是两张页表。所谓“共享虚拟地址”不是
让两张页表自动保持一致。

### 6.3 HMM 快照与失效序号

[`amdgpu_hmm_range_get_pages()`](../../linux/drivers/gpu/drm/amd/amdgpu/amdgpu_hmm.c:168)
的核心顺序是：

```text
mmu_interval_read_begin()
  -> hmm_range_fault()
  -> ...
  -> mmu_interval_read_retry()
```

调用方只有在 retry 检查通过后，才能确认 `hmm_range_fault()` 得到的 PFN/页表
快照没有被 CPU 页表变化打断。

[`svm_range_validate_and_map()`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_svm.c:1731)
把完整顺序写成：

1. reserve GPU page-table/BO；
2. `hmm_range_fault()` 取页地址；
3. 对 system memory page 做 DMA map；
4. 持有 notifier/range 保护；
5. retry 检查页面和 range 仍有效；
6. 更新 GPU PTE；
7. 释放 reservation。

失败时返回 `-EAGAIN`，由 fault/restore policy 重试。这里的关键不是“把 mmap
锁一直持有”，而是保证“页表快照序号未失效”且 GPU PTE 更新与 invalidation
之间的顺序正确。

### 6.4 System page 到 GPU PTE

[`svm_range_dma_map_dev()`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_svm.c:160)
对普通 system page 调用 `dma_map_page(..., DMA_BIDIRECTIONAL)`。对
device-private page，则根据 HMM PFN、VRAM base 和 page owner 生成 VRAM
地址。

这再次说明同一个 SVM range 内可能有不同 residency：

```text
SVM VA --> system page --> DMA address --> GPU PTE
        \-> device page --> VRAM offset --> GPU PTE
```

不能因为两种 PTE 都对应同一个 Guest VA，就认为访问成本、cache 属性或故障
行为相同。

### 6.5 XNACK 与 recoverable fault

[`kfd_ioctl_set_xnack_mode_args`](../../linux/include/uapi/linux/kfd_ioctl.h)
说明 XNACK 让 GFXv9+ 的地址翻译可以返回 retry，使 GPU 在 PTE 暂时无效时
等待或重试。它改变了 SVM 的恢复模型，也要求 shader 使用匹配的编译模式。

KFD fault path 只有在进程启用 XNACK 时才处理 GPU retry fault。源码还会：

- 查找 PASID、process 和 range；
- 检查 VMA 权限和操作是否允许；
- 选择 restore location；
- 必要时在 VRAM/system memory 间迁移；
- 重建 GPU mapping；
- 对 stale fault、重复 fault 和 teardown 做过滤。

因此“有 recoverable fault”不等于任意时刻都可以撤销 PTE，也不等于长 compute
可以无条件被抢占。

### 6.6 MMU notifier 与 CPU 修改页表

[`svm_range_cpu_invalidate_pagetables()`](../../linux/drivers/gpu/drm/amd/amdkfd/kfd_svm.c:2688)
在 invalidate 时更新 notifier sequence，并根据 XNACK 和 range 属性选择：

- 没有 GPU retry fault 能力时，quiesce queue，撤销 GPU 访问，稍后 restore；
- 有 retry fault 能力时，unmap GPU range，让后续 fault 恢复；
- `MMU_NOTIFY_UNMAP` 时撤销 GPU 映射并安排 range 删除/拆分收尾。

复杂路径用序列检查和重试解决竞态。仅靠一个驱动私有大锁会阻塞 eviction，
甚至导致 livelock，这也是
[`gpusvm.rst`](../../linux/Documentation/gpu/rfc/gpusvm.rst) 明确列出的设计
原则。

### 6.7 `drm_gpusvm` 的逐设备 DMA 映射

[`drm_gpusvm_get_pages()`](../../linux/drivers/gpu/drm/drm_gpusvm.c:1771) 可以先对
一个 CPU range 执行 HMM fault，然后为多个 GPU 分别建立 DMA mapping。每个
GPU 都有自己的 `drm_gpusvm_pages` 实例，因此：

- 同一个 CPU page 可以有多个 device-specific DMA address；
- 一个 GPU 的 mapping 失效不一定代表另一个 GPU 也已撤销；
- N:1 range 共享 notifier，但 DMA 映射仍要逐设备管理；
- driver 必须先 unmap DMA，再释放 page 引用。

多 GPU SVM 文档因此不能把“建立了一次 HMM fault”误写成“只维护一个全局 PTE”。

## 7. 迁移与 eviction

### 7.1 迁移不是简单 memcpy

[`hmm.rst`](../../linux/Documentation/mm/hmm.rst:315) 把 migration 描述为：

```text
lock source pages / isolate from LRU
  -> install migration PTE
  -> allocate destination
  -> invalidate device mappings as required
  -> copy
  -> commit CPU page-table replacement
  -> update device PTE
  -> finalize and unlock
```

驱动拥有“迁到哪里”和“何时迁”的 policy；核心 MM 提供页表、migration PFN 和
并发控制基础。

### 7.2 `drm_pagemap` 迁到 device memory

[`drm_pagemap_migrate_to_devmem()`](../../linux/drivers/gpu/drm/drm_pagemap.c:668)
在持有 mmap lock 的前提下：

1. 选择系统页、device-coherent/private page；
2. `migrate_vma_setup()` 锁定并隔离源页；
3. 由 driver 提供目的 device PFN；
4. DMA map 后调用 `copy_to_devmem()`；
5. `migrate_vma_pages()` 提交迁移；
6. 检查竞态并 `migrate_vma_finalize()`。

源码还会处理同一 device pagemap 的页、复合页和部分被其他路径已迁移的页。

### 7.3 `drm_pagemap` eviction 回 RAM

[`drm_pagemap_evict_to_ram()`](../../linux/drivers/gpu/drm/drm_pagemap.c:1313) 是
无 CPU VA 时释放 device memory 的路径：

1. 用 `populate_devmem_pfn()` 得到 device-private PFN；
2. `migrate_device_pfns()` 隔离 device page；
3. 为 system page 建立 DMA mapping；
4. `copy_to_ram()`；
5. `migrate_device_pages()` 和 `migrate_device_finalize()`。

它不要求 mmap lock，因为它只依赖物理页和 device migration 基础设施，不通过
VMA 反查 CPU VA。GPU PTE 的失效由拥有 GPU VA 的 notifier 路径处理。

这与 `gpusvm.rst` 的原则一致：CPU VA 可能因 `mremap()` 等操作改变，不能用
physical page 中保存的 CPU VA 反向推导当前映射。

### 7.4 数据从哪里迁到哪里

AI workload 常见迁移方向：

| 方向 | 触发者 | 典型原因 |
| --- | --- | --- |
| RAM -> VRAM | first touch、prefetch、GPU page fault | kernel 即将访问 |
| VRAM -> RAM | CPU fault、eviction、压力 | CPU 要访问或腾出 HBM |
| GPU A -> GPU B | explicit policy、fault、collective | data locality |
| RAM -> peer memory | P2P/HMM policy | 大模型分片 |
| VRAM -> swap/checkpoint | runtime | 容量管理和恢复 |

每次迁移都要问：

- 原 GPU PTE 何时失效和 flush；
- 新 PTE 何时对所有 GPU 可见；
- 迁移失败时旧映射如何回滚；
- CPU page 是否被标记 dirty；
- 正在使用旧 backing 的 compute job 是否可被 preempt/retry。

## 8. CPU/GPU coherence 的完整判断

### 8.1 DMA map 不是 coherence 的同义词

`dma_map_page()`/`dma_map_sgtable()` 建立 DMA address，并按方向和平台需要执行
cache maintenance。它不等于设备在任意时刻自动观察 CPU cache，也不等于 CPU
可以无同步地读取设备 cache。

[`DMA_ATTR_REQUIRE_COHERENT`](../../linux/Documentation/core-api/dma-attributes.rst:167)
要求映射在需要 SWIOTLB 或显式 cache management 的平台上失败。它适合表达
“该 uAPI 需要持续硬件 DMA coherence”的要求，而不是所有 GPU mapping 的默认
属性。

[`DMA_ATTR_SKIP_CPU_SYNC`](../../linux/Documentation/core-api/dma-attributes.rst:46)
则允许调用者跳过 cache sync，但调用者必须已经证明数据位于正确 domain。
把两者混用会把性能优化变成数据损坏。

### 8.2 GPU PTE 属性

[`amdgpu_vm.h`](../../linux/drivers/gpu/drm/amd/amdgpu/amdgpu_vm.h:59) 可见
`AMDGPU_PTE_SNOOPED`、`AMDGPU_MTYPE_CC`/`NC` 等概念。其含义依赖 GPU
generation、memory type、连接拓扑和系统 cacheability。

PTE 中的 snoop/mtype 表达硬件访问策略，但它不能修复：

- 错误的 DMA direction；
- 迁移期间的 stale PTE；
- 漏掉的 GPU TLB invalidate；
- API 层缺失的 barrier/fence；
- guest/host 对同一页的不同 cacheability 假设。

### 8.3 三类“同步”不要混淆

| 同步对象 | 证明什么 | 不证明什么 |
| --- | --- | --- |
| ring head/tail | 控制命令是否被消费 | GPU work 是否完成 |
| dma_fence/VkFence | 指定 queue work 的完成顺序 | 所有写者自动可见 |
| API barrier/cache flush | 指定资源在 API 语义下可见 | 跨 VM 的所有映射都有效 |

正确的数据可见性通常是：

```text
address mapping valid
  + residency stable
  + cache/ordering operations complete
  + completion/fence observed
  = data may be consumed by the other engine
```

缺少任意一项都不能由另外一项替代。

### 8.4 Coarse-grained 与 fine-grained

ROCm/HIP 和部分 GPU runtime 会区分：

- coarse-grained：在显式同步点之后保证系统与 GPU 的可见性；
- fine-grained：更频繁地要求系统与 GPU 之间具有一致性；
- extended/device-scope coherent：原子和跨设备一致性范围更大。

虚拟化不能只看 Guest API 的 flag。Host 为 GPU 建立的 PTE、IOMMU/平台能力、
Host driver 的 DMA policy 和 fence 都必须支持同一语义。

因此一个 userptr 实验即使没有出现数据错误，也不能仅凭一次运行证明所有平台
都提供连续 coherence。

## 9. 硬件与软件虚拟化方案的内存边界

IOMMUFD、PASID/PRI、ATS、ACS 和 P2P 的物理基础设施层见
[PCIe、IOMMU、ATS/PASID 与 GPU P2P](pcie-iommu-ats-p2p.md)。

### 9.1 Virtio-GPU API remoting

```text
Guest application
  -> Guest UMD
  -> Guest virtio-gpu KMD
  -> resource/blob/ring
  -> QEMU/renderer
  -> Host graphics API or native driver
```

Guest 不拥有物理 GPU BAR。内存能力来自 resource、blob、dma-buf、mmap window
和协议 fence。隔离主要依赖：

- QEMU 对 descriptor/resource/offset/size 的验证；
- renderer 协议解析和对象表；
- Host driver/IOMMU 对 renderer 进程的保护；
- Guest 与 Host resource 的生命周期管理。

它适合跨厂商、API remoting 和软件中介共享；代价是 API/协议兼容性以及
Host parser 性能。

### 9.2 Virtio-GPU native DRM context

Guest 使用接近 DRM 的 context/UAPI，Host renderer 再调用 Host native GPU
driver。Guest 仍不直接拥有物理 PCI GPU。

与 VirGL/Venus 相比：

- 命令语义更接近 GPU driver UAPI；
- Guest KMD/UMD 仍需要 VirtIO transport；
- native resource/dma-buf/VA 语义耦合 Host driver 版本；
- 隔离仍主要是软件 host driver + IOMMU，而不是独立的 PCIe VF。

它更像“DRM UAPI remoting/mediation”，不是 VFIO passthrough。

### 9.3 VFIO 整设备直通

Guest 获得物理 GPU 的 PCI identity、BAR 和 MSI-X，并运行原生 Guest GPU
driver。设备的 DMA 请求必须经过 Host IOMMU，映射到该 Guest 被允许的 GPA。

```text
Guest GPU driver
  -> physical GPU MMIO
  -> GPU VA
  -> GPU page table
  -> Guest-system-memory GPA
  -> IOMMU
  -> Host HPA
```

VRAM 由该 Guest 独占整个设备时通常也在该租户的单一 GPU VM 中。IOMMU 限制
DMA 范围，但不会自动解决：

- GPU firmware/command processor 漏洞；
- reset 对共享 PCIe 拓扑的影响；
- 热迁移不支持的 VRAM；
- 设备自身 DMA bypass/ATS 配置错误。

### 9.4 SR-IOV VF

每个 VF 有独立 PCIe function、BAR、中断和 Guest 原生 driver。PF 负责创建和
管理 VF，硬件为 VF 提供 GPU VM、queue 和资源切分。

内存边界通常是：

- VF 自己的 VRAM/HBM quota；
- VF 的 GPU 页表；
- Guest system memory 仍经 Host IOMMU；
- 是否允许多个 VF P2P 由硬件拓扑/ACS/IOMMU policy 决定。

VF 是 PCIe 设备模型，不只是软件 quota。但“有 VF”仍不自动证明所有硬件
engine、SR-IOV live migration 或 fault reset 都被完整隔离。

### 9.5 mdev

mdev 是 Host kernel 向 VMM 暴露 mediated virtual device 的框架。厂商驱动
定义 carveout、兼容性和生命周期。Guest 可能使用厂商虚拟 GPU driver，关键
MMIO/Memory access 被 Host driver 截获或转发。

内存不天然是 Guest 物理独占的 VRAM：

- Host driver 维护多个 Guest 的资源；
- Guest aperture/BAR 可能由软件模拟；
- 共享 system memory 通过 trap、mmap、dma-buf 或 vendor 协议；
- 隔离保证取决于 vendor driver 和硬件。

mdev 解决“没有可用 SR-IOV VF 时如何多租”，不保证等同于 VF 的硬件边界。

### 9.6 Mediated Pass-Through（MPT）

MPT 把性能关键资源（queues、doorbell、部分 MMIO、显存窗口）直接分配给
Guest，把控制路径（配置、调度、reset、迁移）留在 Host。

内存上通常组合：

- Guest 自己的 GPU VM/queue memory；
- IOMMU 限制 system-memory DMA；
- GPU 内部 scheduler/partition 限制 VRAM 和 engine；
- trap 模拟敏感的 BAR/MMIO 访问。

MPT 不是单一 Linux 子系统名；它是一类半直通架构。不能仅凭某个设备使用
mdev 或 VF，就断言它一定是 MPT。

### 9.7 NVIDIA MIG

MIG 在单个物理 GPU 内把 SM/L2/DRAM/memory slice 等资源划分成实例。它不是
PCIe SR-IOV VF，也不等于完整 vGPU 栈。

一个 MIG instance 可以被 platform virtualization/assignment 使用，但：

- GPU 级 PCIe/config/reset 边界仍可能共享；
- profile 决定 compute/memory 能力；
- P2P、媒体 engine 和 MIG 组合受硬件/驱动限制；
- HBM capacity 和带宽通常是分区，不等于“整卡内存加 quota flag”。

### 9.8 厂商 vGPU

vGPU 是产品级术语，通常包含：

- Guest 厂商 GPU driver；
- 虚拟 PCI function/BAR；
- Host 调度、VRAM accounting 和 engine sharing；
- 平台/VMM 管理接口；
- 部分实现建立在 SR-IOV 或硬件 partition 上。

其内存语义取决于具体产品代际，不能用一个通用 PTE 模型替代实测/官方约束。
共同点是 Guest 看到自己的 GPU aperture、显存容量和 queue/context，Host
负责跨 VM 调度和隔离。

### 9.9 横向比较

| 方案 | Guest 设备模型 | GPU 内存主边界 | 命令执行边界 | 是否等于物理独占 |
| --- | --- | --- | --- | --- |
| virtio-gpu remoting | virtio GPU | Guest resource/blob；Host renderer 决定实际 backing | Host API/native renderer | 否 |
| VFIO | 物理 PCI function | 整设备 VRAM + Guest RAM/IOMMU | Guest 原生 GPU driver | 通常整设备 |
| SR-IOV VF | PCIe VF | VF VRAM/HBM partition + IOMMU | Guest 原生 driver 驱动 VF | 资源级隔离 |
| mdev | mediated virtual device | Host vendor driver 管理/共享 | Host interception | 否 |
| MPT | 半直通 virtual/partial device | 直接窗口 + Host 控制 | Guest fast path + Host control | 依设计 |
| MIG | GPU internal instance | SM/L2/HBM slice | Host/VM assignment + partition | 内部资源分区 |
| vendor vGPU | virtual GPU function | Guest VRAM quota + Host accounting | Host scheduler + guest driver | 依产品 |

## 10. 生命周期、失败路径与安全检查

### 10.1 推荐的对象状态机

对每个 resource/backing 至少记录：

```text
created
  -> mapped
  -> resident (optional)
  -> referenced by submitted work
  -> work complete
  -> unmap
  -> unpin/free
```

这些状态不能由单个引用计数替代。一个 allocation 可能仍然有句柄，但 GPU PTE
已因 notifier 失效；也可能 GPU work 已完成，但 CPU 尚未同步 dirty page。

### 10.2 撤销顺序

一般安全方向是：

1. 阻止新命令引用 backing；
2. 等待或终止使用旧 mapping 的 GPU work；
3. invalidate/flush GPU TLB；
4. DMA unmap；
5. 撤销 CPU/IOMMU mapping 或迁移 residue；
6. 释放 page/object 引用；
7. 最后释放 driver tables/context。

不同架构的顺序可能不同，但任何跳过“停止旧访问”或“GPU TLB invalidate”的
实现都值得怀疑。

### 10.3 典型失败

#### UAF during unmap

Host/GPU 仍持有旧 Host VA 或 DMA address，Guest 页已释放。

#### Stale GPU PTE

CPU 页表或 residency 已改变，但 GPU PTE 尚未 invalidate/rebuild，设备访问
旧物理页。

#### Coherence mismatch

DMA mapping 成功，但平台使用 bounce buffer 或需要显式 cache maintenance，
而 API 被错误描述为 continuous coherent。

#### Pin/memlock exhaustion

userptr 长期 pin 大量页，造成 Guest 内存压力、OOM 或租户 DoS。

#### Migration livelock

CPU 与 GPU 持续触发同一页反向迁移。`gpusvm.rst` 将驱动私有锁描述为风险，
并强调 retry/migration policy 需要避免无限重试。

#### Long compute preemption

[`drm-compute.rst`](../../linux/Documentation/gpu/drm-compute.rst) 明确说明：
pin 全部 compute buffer 会导致内存 DoS；没有 preemption/recoverable fault 时，
unbind 可能只能终止 long-running job。生产多租户系统需要 cgroup-aware
accounting/eviction 或等价机制。

### 10.4 安全审查

必须验证：

- Guest 是否能控制资源 ID、size、offset 和 DMA address 的非法组合；
- resource destroy 是否覆盖所有 failed-path；
- map/unmap 是否与 pending command 有序；
- userptr 的 munmap/mremap/process exit 是否安全；
- GPU fault 是否能被恶意地址放大为 Host OOM；
- 一个租户的 fence/queue error 是否影响其他租户；
- MIG/VF/vGPU 的 memory quota 是否只统计 allocation，还是也统计迁移/临时
  staging；
- checkpoint/reset 时是否仍有对外可见的 backing。

## 11. AI workload 的内存问题

### 11.1 容量不是唯一指标

模型训练/推理需要同时考虑：

| 资源 | 示例 |
| --- | --- |
| HBM capacity | weights、activation、gradient、optimizer、KV cache |
| HBM bandwidth | GEMM、attention、memory-bound kernel |
| GTT/system memory | staging、host overflow、SVM |
| pinned memory | async copy、DMA、userptr |
| migration bandwidth | page fault、eviction、checkpoint |
| allocator fragmentation | long-lived tensors + dynamic activation |

因此“给 Guest 分配 24 GiB VRAM quota”不足以描述性能或安全性。还要说明：

- quota 是 hard cap 还是可 eviction；
- page table/context/workspace 是否计入；
- 迁移和压缩是否使用同一 quota；
- OOM 时终止进程、返回 allocation failure，还是降低 occupancy。

### 11.2 KV cache 与长生命周期

LLM decode 的 KV cache 会随着 sequence/batch 增长，常驻 HBM。其特点是：

- allocation 时间长；
- 频繁小更新；
- 容量增长可预测但峰值高；
- 请求结束才释放；
- eviction 到 CPU 可能直接破坏 decode 尾延迟。

这类 workload 适合显式容量/分区和稳定映射，不适合每轮重新 fault/migrate。

### 11.3 Checkpoint 与迁移

checkpoint 需要保存：

- weight、optimizer、gradient；
- RNG/scheduler/optimizer step；
- collective communicator 状态；
- device memory resident/in-flight 状态；
- page table、queue、event 和 fence。

只复制 HBM bytes 不够。若 Host/GPU 仍在写页面，必须先建立稳定快照或按协议
暂停。live migration 还要处理目的地 GPU 架构、PTE 属性、dma-buf 和 topology
兼容性。

### 11.4 多 GPU 与 P2P

多 GPU 路径需要区分：

```text
GPU A VA -> local GPU MMU -> peer GPU memory
GPU A VA -> local GPU MMU -> system memory -> peer GPU / NIC
```

IOMMU、PCIe ACS、NVLink/XGMI topology、VF/MIG peer 限制和 Host scheduler
都可能改变 P2P。NCCL/RCCL 的 topology detection 只能选择它认为可用的路径；
如果虚拟化隐藏或错误暴露了 P2P 能力，collective 会退化或失败。

## 12. 从症状定位到正确层

| 症状 | 先查 | 再查 |
| --- | --- | --- |
| 数据全是旧值 | API barrier/fence | cache maintenance、PTE cache 属性 |
| GPU page fault | VMA/range 是否注册 | HMM fault、migration、GPU PTE |
| CPU 访问触发迁移 | device-private PTE | `migrate_to_ram`、notifier seq |
| munmap 后 device lost | queue quiesce/TLB invalidate | DMA unmap 和 resource ref |
| map 成功但内容错 | map_info/cache type | dma-buf modifier/format |
| read 正常 write 错 | PTE writable/FOLL_WRITE | GPU cache flush 和 ownership |
| 小数据正常大数据错 | SG/IOMMU/compound page | migration retry、range split |
| 单卡正常 collective 差 | topology/P2P | IOMMU/ACS、virtual network |
| 训练偶发 OOM | pin/quota/allocator | migration staging 和 cgroup accounting |
| fence 已完成但数据仍错 | fence 覆盖范围 | barrier、cache flush、resource mapping |

这张表不是故障实验的替代品，而是决定先收集哪类证据的起点。

## 13. 方案评审模板

评审一个 GPU 虚拟化或 AI 内存方案时，至少填写：

```text
Address domains:
  guest VA / GPA / host VA / IOVA / GPU VA / VRAM offset

Allocation:
  owner, allocator, quota, accounting, OOM policy

Mapping:
  CPU page table, IOMMU, GPU page table, map granularity

Residency:
  RAM/VRAM/peer/staging; migration and eviction policy

Coherence:
  DMA attributes, GPU MTYPE/snoop, API barrier, cache flush

Completion:
  queue fence, TLB invalidate, timeline, WSI acquire/release

Lifetime:
  object ref, pin, notifier, pending work, teardown order

Isolation:
  software parser, IOMMU, PCIe VF, GPU partition, mediated driver

Failure:
  page fault, OOM, reset, process exit, migration rollback
```

只有这些字段都有明确答案，才能讨论“零拷贝”“共享显存”“可迁移”或
“接近原生”等结论。

## 14. 检查题

1. 为什么 Guest CPU VA、GPA、QEMU Host VA、IOVA 和 GPU VA 不能互换？
2. `virtio_gpu_object_shmem_init()` 为什么可能写 DMA address，也可能写
   `sg_phys()`？
3. `VIRTGPU_BLOB_MEM_GUEST`、`HOST3D` 和 `HOST3D_GUEST` 的所有权差别
   是什么？
4. 为什么 mappable ring blob 的映射不能证明任意 Vulkan/HIP allocation 都是
   zero-copy？
5. userptr 的 long-term pin 解决了什么，又带来了什么 DoS 和 teardown 风险？
6. `dma_map_sgtable()` 成功后为什么仍可能不满足 fine-grained coherence？
7. `mmu_interval_read_begin()`、`hmm_range_fault()` 和
   `mmu_interval_read_retry()` 分别承担什么作用？
8. `drm_pagemap_migrate_to_devmem()` 和 `drm_pagemap_evict_to_ram()` 为什么
   使用不同的 migration 接口和锁前提？
9. XNACK 如何改变 SVM invalidation/fault 恢复模型？
10. GPU PTE 的 snoop/MTYPE、DMA API cache maintenance 和 API barrier 分别
    解决哪一层问题？
11. Virtio-GPU native DRM context、VFIO、SR-IOV 和 MIG 分别由谁建立
    Guest-visible GPU 内存边界？
12. 为什么 MIG instance 不等于 SR-IOV VF，也不能自动推出完整 vGPU/VM
    迁移能力？
13. KV cache 和训练 checkpoint 对 migration、quota、fence 的要求为什么不
    同？
14. 一个长 compute job 在什么条件下可以安全撤销 GPU PTE，什么条件下只能
    终止或等待？

## 15. 推荐源码顺序

第一轮，建立地址和对象模型：

1. `linux/include/uapi/drm/virtgpu_drm.h`
2. `linux/drivers/gpu/drm/virtio/virtgpu_object.c`
3. QEMU `virtio_gpu_create_mapping_iov()`
4. virglrenderer `virgl_renderer_resource_create_blob()`

第二轮，理解 SVM：

1. `linux/Documentation/mm/hmm.rst`
2. `linux/Documentation/gpu/rfc/gpusvm.rst`
3. `linux/drivers/gpu/drm/drm_gpusvm.c`
4. `linux/drivers/gpu/drm/drm_pagemap.c`
5. `linux/drivers/gpu/drm/amd/amdgpu/amdgpu_hmm.c`
6. `linux/drivers/gpu/drm/amd/amdkfd/kfd_svm.c`

Intel Xe 的逐函数 fault/userptr/migration 跟踪见
[Intel Xe SVM、Userptr 与 GPU Page Fault 源码跟踪](intel-xe-svm-source-trace.md)。

dma-buf attachment/DMA mapping/reservation/fence/sync_file 的逐函数语义见
[DMA-BUF、DMA-FENCE、Reservation 与 Sync File 源码跟踪](dma-buf-fence-source-trace.md)。

第三轮，理解虚拟化边界：

1. [Virtio-GPU Userptr 与 ROCm compute](../virtio-gpu/userptr-rocm-compute.md)
2. [ROCm compute 源码跟踪](../virtio-gpu/rocm-source-trace.md)
3. [AMD KFD/ROCr 源码跟踪](amdkfd-source-trace.md)
4. [硬件/软件/API 虚拟化方案比较](../virtio-gpu/platform-comparison.md)
5. [NVIDIA CUDA、MIG 与 vGPU](nvidia-cuda-vgpu.md)
