# master of orw

## 题目简述

程序分配一段最多 `0x400` 字节的可执行内存，读入用户 Shellcode，加载 seccomp 后跳转执行。常规 ORW 所需的 `open/openat`、`read`、`write` 以及 `execve` 等系统调用均被禁止，但 `io_uring_setup`、`io_uring_enter` 和 `mmap` 仍可使用。

目标是不用被禁的系统调用指令直接发起文件操作，而是通过 io_uring 的共享队列提交 `OPENAT → READ → WRITE` 三个异步请求，读取当前目录下的 `flag`。

## 解题过程

### 1. 找到 seccomp 黑名单之外的 I/O 接口

seccomp 检查的是进程实际进入内核时使用的系统调用号。io_uring 的接口不同：用户态先用 `io_uring_setup` 创建共享环形队列，通过 `mmap` 映射 SQ、CQ 和 SQE 数组，把文件操作描述成 SQE，最后调用 `io_uring_enter` 通知内核处理。

[io_uring(7)](https://man7.org/linux/man-pages/man7/io_uring.7.html) 对这一模型的关键说明是：用户把操作写进 submission queue，内核异步执行后把结果写进 completion queue；提交请求时直接进入内核的是 `io_uring_enter`，SQE 中才包含等价文件操作的参数。本题已允许这些基础系统调用，因此可以在题目内核上绕过只按 `open/read/write` 号设置的黑名单。

这不是所有内核和所有沙箱上的通用结论。正确审计方式是检查实际 seccomp 规则、内核版本、io_uring 是否可创建，以及目标 opcode 是否受额外限制；这里只讨论附件对应的部署环境。

### 2. 建立最小 io_uring

官方 exp 没有把完整 liburing 静态链接进 Shellcode，而是从库实现中抽取了题目所需的最小逻辑。x86-64 上直接使用：

```text
io_uring_setup = 425 (0x1a9)
io_uring_enter = 426 (0x1aa)
mmap           =   9
```

初始化步骤为：

1. 在栈上清零 `struct io_uring_params`，以 16 个队列项调用 `io_uring_setup`；
2. 根据返回的 offsets 映射 SQ/CQ ring；
3. 以偏移 `IORING_OFF_SQES = 0x10000000` 再映射 SQE 数组；
4. 保存 SQ/CQ 的 head、tail、mask、array 与 CQE 指针，后续直接读写共享内存。

获取 SQE 时，用本地 tail 计算槽位：

```text
index = sqe_tail & ring_mask
sqe   = sqes + index * 64
```

填好 SQE 后，把索引写入 SQ array，以 release 语义推进 SQ tail，再调用 `io_uring_enter(ring_fd, 1, ...)` 提交。Shellcode 只有一个执行线程，但共享队列仍要求正确的发布顺序；完整 C 实现应使用 liburing 或原子屏障，不能把普通内存写顺序当成跨架构保证。

### 3. 用 SQE 完成 ORW

三种操作的 opcode 是：

```text
IORING_OP_OPENAT = 18 (0x12)
IORING_OP_READ   = 22 (0x16)
IORING_OP_WRITE  = 23 (0x17)
```

官方 exp 在栈上写入立即数 `0x67616c66`。x86-64 为小端序，对应内存字符串 `flag\x00`。第一个 SQE 的核心字段等价于：

```c
io_uring_prep_openat(sqe, AT_FDCWD, "flag", O_RDONLY, 0);
```

其中 `AT_FDCWD` 为 `-100`，即 32 位补码 `0xffffff9c`。提交后，打开得到的文件描述符不是系统调用返回值，而是第一个 CQE 的 `res` 字段。取出该 fd 后，继续提交：

```c
io_uring_prep_read(sqe, fd, buffer, 0x100, 0);
io_uring_prep_write(sqe, STDOUT_FILENO, buffer, 0x100, 0);
```

最后输出 CQE 中实际读取的字节数更严谨；官方 exp 固定写 `0x100`，因此 flag 后可能带有未使用缓冲区内容。

### 4. 压缩到 `0x400` 字节并保证完成顺序

完整 liburing 和静态 C 程序远超长度限制，因此 exp 手写了以下最小函数：

```text
queue_init → mmap rings → get_sqe → prep_rw → submit → locate_cqe
```

它删除了错误处理、通用 feature 支持、队列退出和非必要字段，只保留附件内核需要的布局。最终发送的是 `asm(shellcode)` 的机器码，而不是整个 ELF。

官方脚本中的 `io_uring_wait_cqe` 实际只根据 CQ head 计算 CQE 地址，没有调用带 `IORING_ENTER_GETEVENTS` 的等待操作；READ 与 WRITE 之间也没有显式链接。它在比赛环境中能够工作，但存在异步时序依赖。更稳健的版本应在每一步：

1. 调用 `io_uring_enter` 时设置 `min_complete = 1` 与 `IORING_ENTER_GETEVENTS`；
2. 确认 `cq_tail != cq_head` 后读取 `cqe->res`；
3. 检查负错误码并推进 CQ head；
4. 等 READ 完成后再按实际长度提交 WRITE，或使用 `IOSQE_IO_LINK` 明确串联请求。

这些同步代码会增加 Shellcode 长度，需要与 `0x400` 上限权衡。官方 exp 在结尾调用 `nanosleep` 给异步 WRITE 留出时间，是竞赛环境下的长度优化，不应当成通用 io_uring 模板。

## 方法总结

本题考查的是“系统调用过滤”与“内核提供的实际能力”之间的差距。黑名单禁止 `open/read/write` 并不等于禁止文件 I/O；只要仍允许创建和提交 io_uring，SQE 就能表达相同操作。

利用链为：创建 ring、映射共享队列、提交 `OPENAT` 并从 CQE 取 fd、提交 READ、再把缓冲区 WRITE 到标准输出。真正困难的是在 `0x400` 字节内重建最小队列协议，同时处理共享内存布局与异步完成顺序。
