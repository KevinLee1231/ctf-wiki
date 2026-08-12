# elite ball knowledge

## 题目简述

程序是静态、非 PIE、未启用栈保护的 x86-64 二进制。`main` 只有 16 字节栈缓冲区，却在安装 seccomp 前执行 `fgets(buf, 0x676700, stdin)`，可以直接覆盖保存的返回地址。随后 seccomp 对编号 $0\ldots335$ 的 syscall 一律返回 `EPERM`，仅放行 `exit` 与 `exit_group`；较新的 syscall 编号则意外保持允许。

因此普通的 `open`、`read`、`write`、`execve` 链不可用。题目的关键不是只拿到 ROP，而是借仍被允许的 `openat2` 和 `io_uring` 让内核代为读写 flag。

## 解题过程

### 确认原语与沙箱空洞

漏洞与过滤顺序如下：

```c
char buf[16];
fgets(buf, 0x676700, stdin);
if (setup_sandbox() != 0) return 1;

for (int i = 0; i <= 335; i++) {
    if (i == 60 || i == 231) continue;
    seccomp_rule_add(ctx, SCMP_ACT_ERRNO(EPERM), i, 0);
}
```

覆盖偏移为 `0x18`（16 字节 `buf` 加保存的 `rbp`）。过滤器遗漏的 syscall 包括：`io_uring_setup=425`、`io_uring_enter=426` 与 `openat2=437`。二进制静态、非 PIE，故官方 solver 可以使用固定 gadget 向 `.bss` 写任意 qword，并调用 libc 的 `syscall(num, a0, ..., a5)` wrapper。

### 通过 io_uring 读取并输出 flag

ROP 链先把以下数据写到固定 `.bss` 区域：

- `"flag.txt\0"` 与全零的 `struct open_how`；
- `IORING_SETUP_NO_MMAP` 的 `io_uring_params`，其 SQE 与 CQ/SQ ring 用户地址也指向 `.bss`；
- 两个 64 字节 SQE、SQ array `{0, 1}` 与 SQ tail `2`；
- 用于文件内容的读缓冲区。

随后依次调用：

```text
openat2(AT_FDCWD, "flag.txt", &how, 24)  -> fd 3
io_uring_setup(8, &params)                -> fd 4
io_uring_enter(4, 2, 2, IORING_ENTER_GETEVENTS, NULL, 0)
exit_group(0)
```

第一条 SQE 是 `IORING_OP_READ`，从 fd 3 读取到 `READ_BUF`；第二条是 `IORING_OP_WRITE`，将同一缓冲区写到 fd 1。第一条附带 `IOSQE_IO_HARDLINK`，所以即使读取长度短于请求长度，写操作仍会执行。核心的 SQE 头部编码为：

```python
def sqe_head(opcode, flags, fd):
    return (opcode & 0xff) | ((flags & 0xff) << 8) | ((fd & 0xffffffff) << 32)

sqe0 = sqe_head(22, 1 << 3, 3)  # READ + IOSQE_IO_HARDLINK
sqe1 = sqe_head(23, 0,      1)  # WRITE stdout
```

这里不需要也不能直接调用被禁止的 `read`/`write`。`io_uring_enter` 让内核按 SQE 执行这两项操作，flag 最终从已有 stdout 输出。官方 solver 得到 `grey{3l1t3_b4lL_kn0wLedge_is_just_more_syscalls}`。

## 方法总结

- 核心技巧：栈 ROP 加上 seccomp syscall 编号范围遗漏，使用 io_uring 代替被禁用的文件 I/O。
- 识别信号：过滤规则只枚举旧编号区间、而不是默认拒绝后按需放行；较新的内核接口应逐个检查。
- 复用要点：`IORING_SETUP_NO_MMAP` 可使 ring/SQE 使用攻击者已有的可写内存，适合 `mmap` 也被限制的环境。构造 SQE 时要同时满足结构体对齐、ring offset、fd 分配顺序和链接语义。
