# EasyVirtio

## 题目简述

题目运行一个经过修改的 QEMU 8.0.0-rc2，只向虚拟机暴露 `virtio-crypto-pci` 设备。基础漏洞是 CVE-2023-3180：`virtio_crypto_sym_op_helper()` 没有验证对称加密请求中的 `src_len` 与 `dst_len` 相等，后端可能按源长度向较短的目标缓冲区写入，造成 QEMU 进程堆溢出。

为了把真实漏洞改造成稳定的比赛利用，题目又加入两个补丁：失败的加解密请求仍会把目标缓冲区复制回 Guest，用于越界读；`CryptoDevBackendSymOpInfo` 尾部放置一个 `g_free` 函数指针，释放请求时改为调用该指针，用于把越界写直接转化为控制流劫持。

## 解题过程

### 1. 理清 VirtIO 请求的数据流

Guest 需要先为 `virtio-crypto` 建立对称加密会话，再通过数据队列提交请求。描述符链同时包含设备可读的请求头、IV、源数据，以及设备可写的目标缓冲区和状态字。QEMU 解析请求中的长度后，一次性分配：

```text
CryptoDevBackendSymOpInfo
data[] = IV | AAD | source | destination | digest | hook
```

记各部分长度为 `iv_len`、`aad_len`、`src_len`、`dst_len` 与 `digest_len`。目标缓冲区起点和题目函数指针的位置分别是：

```text
dst  = data + iv_len + aad_len + src_len
hook = data + iv_len + aad_len + src_len + dst_len + digest_len
```

因此 `dst` 到 `hook` 的距离为 `dst_len + digest_len`。只要令 `src_len` 大于这个距离，任何按 `src_len` 处理 `dst` 的读写都会越过目标缓冲区并到达 `hook`。

上游修复提交 [9d38a843](https://gitlab.com/qemu-project/qemu/-/commit/9d38a8434721a6479fe03fb5afb150ca793d3980) 增加了 `src_len != dst_len` 时立即报错的检查。该链接的关键信息就是漏洞产生于长度不一致，以及正确修复应在分配和后端操作前拒绝此类请求；本题使用的旧版本没有该检查。

### 2. 利用题目补丁完成越界读

原版 `virtio_crypto_sym_input_data_helper()` 在后端返回错误时直接结束，不会把未生成的目标数据送回 Guest。题目删除了这段判断：

```c
/* if (status != VIRTIO_CRYPTO_OK) {
 *     return;
 * }
 */
len = sym_op_info->src_len;
iov_from_buf(in_iov, req->in_num, 0, sym_op_info->dst, len);
```

构造一个后端必然失败、同时满足 `src_len > dst_len` 的请求后，QEMU 仍从 `dst` 开始复制 `src_len` 字节到 Guest 的可写描述符。前 `dst_len` 字节属于目标缓冲区，之后则是越界的堆内容。

题目在 `data + max_len` 处主动写入 `g_free`：

```c
*(uint64_t *)(op_info->data + curr_size) = g_free;
```

所以只要 Guest 提供的接收区足够长，越界结果中会出现一个已知符号的真实地址。用该地址减去本地完全匹配运行库中的 `g_free` 偏移，即可恢复对应库的加载基址；泄露到的相邻堆指针和 QEMU 指针还可用于校验堆布局与 PIE 基址。不能只凭一个疑似指针盲套版本，必须先与附件中的 QEMU 和容器运行库对应。

### 3. 利用原始漏洞完成越界写

对于能够正常执行的对称加密请求，后端按 `src_len` 产生输出，却把结果写到按较小 `dst_len` 描述的 `dst` 区域。令：

```text
src_len > dst_len + digest_len + 8
```

即可让输出覆盖尾部的 8 字节 `hook`。密钥、IV 和明文都由 Guest 控制，因此可以针对所选分组模式反向计算输入，使落在 `hook` 位置的密文字节等于目标地址。实际实现时还要同时满足算法的分组对齐要求，并让 Guest 的可写描述符覆盖完整的返回长度，否则请求会先因描述符长度不足而失败。

释放请求时，题目补丁不再固定调用 `g_free(op_info)`，而是执行：

```c
memset(op_info, 0, sizeof(*op_info) + max_len);
((void (*)(void *))(*(uint64_t *)(op_info->data + max_len)))(op_info);
```

`memset` 恰好停在 `hook` 之前，所以被覆盖的函数指针不会被清零。调用发生时程序计数器来自 `hook`，在 x86-64 System V ABI 下第一个参数寄存器 `RDI` 指向已经清零的 `op_info`。选择 one-gadget、栈迁移或寄存器调整 gadget 时，必须以这一现场为约束，不能把它简单等同于无参数任意函数调用。

### 4. 从 QEMU 控制流到 flag

完整利用顺序如下：

1. 在 Guest 中初始化 PCI/VirtIO 设备、协商特性并建立 virtqueue；
2. 创建对称加密会话，提交失败请求触发越界读；
3. 从回传数据定位 `g_free`，计算运行库基址并确定可用控制流目标；
4. 预计算加密输入，提交成功请求，使越界输出覆盖 `hook`；
5. 请求释放时劫持 QEMU 主机进程的执行流。

容器通过 `chroot /home/ctf ./run.sh` 启动 QEMU，并把 chroot 内的 `/bin/sh` 链接到 `/readflag`。因此最终目标是让 Host 侧 QEMU 执行该程序并回传输出，而不是在 Guest 内读取附件中的 `ACTF{dummy}`；后者只是占位文件。仓库没有提供 Guest 侧 exp，现有证据只能严谨还原到上述利用原语和控制流条件，不能据此虚构某个最终 flag 或未经测试的设备驱动代码。

## 方法总结

本题的关键是区分三层事实：CVE-2023-3180 提供 `src_len != dst_len` 的堆越界写；第一个赛题补丁把错误路径变成从 `dst` 开始、长度为 `src_len` 的越界读；第二个赛题补丁在对象尾部加入可泄露、可覆盖、会被间接调用的 `g_free` 指针。

分析 VirtIO 设备漏洞时，应先画清 Guest 描述符、Host 堆对象及各长度字段之间的对应关系，再计算目标相对 `dst` 的精确距离。控制函数指针只是获得 PC，仍需结合调用参数、清零操作、运行库版本和容器内最终目标完成后半段利用。
