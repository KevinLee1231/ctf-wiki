# unwind

## 题目简述

这是一个 32 位 Windows SEH 与 TEA 结合的逆向题。程序读取 64 字节输入，将前 32 字节加密一次、后 32 字节加密两次，再与内置的 64 字节数组比较。难点是第二次加密没有出现在普通控制流中，而是由异常分发和栈展开重复调用同一个 SEH handler 造成的。

## 解题过程

### 先处理反调试

`DisableDebugEvent` 动态取得 `ZwSetInformationThread`，以 `ThreadHideFromDebugger (0x11)` 隐藏当前线程。调试时可直接 NOP 该调用或让函数提前返回；否则断点异常可能使调试器表现异常。这个补丁只影响调试，不改变 flag 校验算法。

### 识别两段不同的加密次数

第一段逻辑很直接：空指针写触发异常，外层 `__except` 对 `enc[0:32]` 的四个 8 字节块分别执行一次 TEA。

第二段调用 `HomeGrownFrame`，它手工把 `_except_handler` 注册到 x86 SEH 链首：

```asm
push handler
push fs:[0]
mov  fs:[0], esp
int  3
```

`int 3` 触发异常后发生两个阶段：

1. 搜索处理器时，系统先调用 `_except_handler`；它加密 `enc[32:64]`，然后返回 `ExceptionContinueSearch`。
2. 外层编译器生成的 `__except` 接受异常。控制流转入该处理块前，系统执行栈展开，又以 unwind 状态调用途经的 `_except_handler`。该函数没有检查异常标志，于是同一半数据再次被加密。

因此内置数组的前四个 TEA 块需要解密一次，后四个块需要解密两次。每个块依次使用 `DX3906`、`doctor3`、`FUX1AOYUN`、`R3verier` 四个以零补齐到 16 字节的密钥；x86 上源码把字节数组强转为 `uint32_t *`，字节序为小端。

```python
from struct import pack, unpack

MASK = 0xFFFFFFFF
DELTA = 0x9E3779B9

encrypted = bytes([
    0x5a, 0xe3, 0x6b, 0xe4, 0x06, 0x87, 0x02, 0x4f,
    0x43, 0xdf, 0xcd, 0xc1, 0x77, 0x98, 0x6b, 0xdb,
    0x8f, 0x38, 0x43, 0x99, 0xe3, 0x93, 0x22, 0xb5,
    0x23, 0xfd, 0xb0, 0x1c, 0xe5, 0xe3, 0xee, 0xce,
    0x2f, 0x1d, 0xad, 0x2b, 0xa4, 0x15, 0x98, 0xf9,
    0xd8, 0xeb, 0x25, 0xfa, 0x6b, 0x21, 0xb7, 0x72,
    0xb9, 0x03, 0x33, 0x2e, 0xd9, 0x4c, 0xeb, 0x7b,
    0xf5, 0xa7, 0x48, 0xf9, 0x90, 0x9d, 0x38, 0xfc,
])
keys = [b"DX3906", b"doctor3", b"FUX1AOYUN", b"R3verier"]

def tea_decrypt(block, key):
    v0, v1 = unpack("<2I", block)
    k0, k1, k2, k3 = unpack("<4I", key.ljust(16, b"\x00"))
    total = DELTA * 32 & MASK

    for _ in range(32):
        mix1 = (((v0 << 4) + k2) & MASK) \
             ^ ((v0 + total) & MASK) \
             ^ (((v0 >> 5) + k3) & MASK)
        v1 = (v1 - mix1) & MASK

        mix0 = (((v1 << 4) + k0) & MASK) \
             ^ ((v1 + total) & MASK) \
             ^ (((v1 >> 5) + k1) & MASK)
        v0 = (v0 - mix0) & MASK
        total = (total - DELTA) & MASK

    return pack("<2I", v0, v1)

plain = bytearray()
for index in range(8):
    block = encrypted[index * 8:index * 8 + 8]
    rounds = 1 if index < 4 else 2
    for _ in range(rounds):
        block = tea_decrypt(block, keys[index % 4])
    plain.extend(block)

print(plain.decode())
```

输出为：

```text
moectf{WoOo00Oow_S0_interesting_y0U_C4n_C41l_M3tW1c3_BY_Unw1Nd~}
```

## 方法总结

x86 SEH handler 不只可能在“寻找处理器”阶段执行，也可能在栈展开阶段再次执行。逆向异常驱动程序时，应同时跟踪 handler 返回值、外层处理器选择和 unwind 路径；只看反编译器展示的普通调用图，会漏掉本题后半段的第二次 TEA。
