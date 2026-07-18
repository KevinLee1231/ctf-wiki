# EasyDMA

## 题目简述

附件提供一套 QEMU 8.2 虚拟机环境，启动参数同时挂载未经修改的 `virtio-blk-pci` 和自定义 `readflag` 设备：

```text
-drive file=null-co://,if=none,id=mydisk
-device virtio-blk-pci,drive=mydisk,ioeventfd=off
-device readflag
```

`readflag` 有两个关键功能：一是以“分配堆块—读入 flag—释放堆块”的方式把 flag 残留在 QEMU 进程堆中；二是暴露一个可读写的 8 字节 MMIO 寄存器。目标是利用 virtio DMA 路径中的未初始化内存回写，把 QEMU 堆中的 flag 搬运到该寄存器，再由 guest 读取。

## 解题过程

### 从设备组合定位漏洞

`readflag` 先制造已释放但仍含敏感数据的堆块，却没有直接把该堆块暴露给 guest，因此还需要一个 QEMU 进程内的信息泄漏原语。附件同时给出原生 virtio-blk、DMA 和可观测 MMIO 寄存器，这组特征对应 [CVE-2024-8612](https://nvd.nist.gov/vuln/detail/CVE-2024-8612)。该漏洞影响 QEMU 的 virtio-blk、virtio-scsi 和 virtio-crypto 路径：设备向 virtqueue 返回的长度可能大于实际初始化的数据长度，之后的 DMA unmap 会把未初始化的 bounce buffer 内容写回 guest 提供的地址，造成宿主进程内存泄漏。

启动参数中的 `ioeventfd=off` 也很重要：virtio 通知保留在 QEMU 用户态设备模型路径中，便于触发后续的地址空间访问和 MMIO 重入。

### 理解 bounce buffer 回写

正常情况下，virtqueue 描述符中的 GPA 指向 guest RAM，QEMU 可以直接映射或把数据复制到临时缓冲区。若 GPA 落在不能直接映射的 I/O 地址空间，例如另一个设备的 MMIO 区域，QEMU 会分配 bounce buffer 暂存数据。

漏洞路径的问题是“声明完成长度”与“实际写入长度”不一致：

1. QEMU 为描述符分配 bounce buffer；
2. virtio 请求只初始化其中一部分，剩余区域仍保留旧堆内容；
3. `virtqueue_push()` 以过大的长度结束请求；
4. `dma_memory_unmap(..., DMA_DIRECTION_FROM_DEVICE, access_len)` 按该长度回写；
5. 如果描述符 GPA 指向 `readflag` 的 MMIO 地址，回写会重入其 MMIO write 回调；
6. 未初始化的 8 字节因此被保存进 `readflag` 寄存器，guest 随后可通过 MMIO read 取回。

这不是传统的“设备把 flag 主动 DMA 给 guest”，而是先让 QEMU 的堆分配器复用含 flag 的已释放堆块，再借错误长度把未初始化内容送进一个可观测设备寄存器。

### 利用流程

完整利用按以下顺序进行：

1. 访问 `readflag` 的控制接口，多次触发其分配、读取并释放 flag 的逻辑，使目标内容进入相应大小的空闲堆块；
2. 在 guest 中构造 virtio-blk 描述符链，把用于设备写回的描述符 GPA 设置为 `readflag` 的 8 字节 MMIO 寄存器地址；
3. 提交能进入 CVE-2024-8612 错误完成长度路径的请求，使新分配的 bounce buffer 复用含 flag 的堆块；
4. 请求完成时，`dma_memory_unmap()` 把未初始化数据写向 MMIO，触发 `readflag` 的 write 回调；
5. 从同一寄存器执行 MMIO read，得到泄漏的 8 字节；
6. 调整堆布局并重复触发，拼接各个 8 字节片段，直到恢复完整 flag。

实际调试时应先确认 virtqueue 的 descriptor、avail ring、used ring 结构和地址端序，再观察 `readflag` 寄存器是否随请求完成发生变化。若只能泄漏零或稳定垃圾，通常是 bounce buffer 没有复用到 flag 堆块、描述符方向设置错误，或完成长度没有进入漏洞分支。

## 方法总结

本题把三个原语串成了一条信息泄漏链：`readflag` 负责把秘密残留在已释放堆中，CVE-2024-8612 负责把未初始化堆内容写出，MMIO 寄存器负责让 guest 观测结果。分析 QEMU 设备题时，应同时跟踪 virtqueue 描述符方向、GPA 所属地址空间、bounce buffer 生命周期以及 DMA unmap 的实际回写长度；只审计单个设备的 MMIO 回调，无法看到这类跨设备 DMA 重入漏洞。
