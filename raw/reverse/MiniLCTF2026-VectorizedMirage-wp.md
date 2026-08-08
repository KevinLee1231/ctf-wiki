# VectorizedMirage

## 题目简述

附件是 Windows 原生程序，校验被拆成明文 TEA 和基于 VEH（Vectored Exception Handling）的隐藏 VM 两段。程序要求 `miniL{...}` 前缀，并用 `IsDebuggerPresent()` 做简单反调试。花括号内的前 16 字节由普通函数做两个 TEA 分组校验，接下来 40 字节则在 `__debugbreak()` 引发的 VEH handler 链中做五个 XXTEA-like 分组校验。

程序实际处理 `miniL{` 之后的 56 字节，其中第二段的最后一字节已经是 `}`，所以标准 flag 总长度是 $6+56=62$ 字节。决定性难点是注册在全局初始化阶段的大量 VEH handler：单个 handler 只表示栈操作、移位或逻辑运算，必须按异常处理链的顺序重组出真实算法。

## 解题过程

### 第一段：逆转标准 TEA

`main` 将 `input + 6` 处的前 16 字节拆成两个 64-bit 小端分组，使用固定 key：

```text
K = 11223344 55667788 99aabbcc ddeeff00
delta = 9e3779b9
rounds = 32
```

目标密文为：

```text
5c6c723e ef5298d9 62d93d11 098a4e7f
```

按 TEA 标准顺序从 `sum = 32 * delta` 向后逆 32 轮即可：

```python
MASK = 0xFFFFFFFF


def tea_decrypt(block, key):
    v0, v1 = block
    delta = 0x9E3779B9
    total = delta * 32 & MASK
    for _ in range(32):
        v1 = (
            v1 - (((v0 << 4) + key[2]) ^ (v0 + total) ^ ((v0 >> 5) + key[3]))
        ) & MASK
        v0 = (
            v0 - (((v1 << 4) + key[0]) ^ (v1 + total) ^ ((v1 >> 5) + key[1]))
        ) & MASK
        total = (total - delta) & MASK
    return v0, v1
```

两个分组解出 body 的前 16 字节：

```text
EASY_UPX_AND_TEA
```

### 第二段：恢复 VEH 虚拟指令链

全局初始化函数 `before_main()` 通过 `AddVectoredExceptionHandler` 注册大量 handler。绝大多数 handler 执行一个小操作后返回 `EXCEPTION_CONTINUE_SEARCH`，让同一次 breakpoint 异常继续沿链传递；末尾 `VmEnd` 跳过 `int3` 并返回 `EXCEPTION_CONTINUE_EXECUTION`。

虚拟机维护 `R_V0`、`R_V1`、`R_SUM`、`R_Y`、`R_Z`、`R_E` 和一个 32-bit stack。为了隐藏异或，程序使用如下门级等价式：

```text
A XOR B = NOT(A AND B) AND NOT((NOT A) AND (NOT B))
```

需特别以 handler 的真实实现为准，不能相信函数名中的数字：

```text
VmShr5 -> >> 2
VmShl2 -> << 5
VmShr3 -> >> 4
VmShl4 -> << 3
VmShr2 -> >> 3
```

重组 `EMIT_XXTEA_ROUND` 后可知，第二段每 8 字节为一组，`n=2`，执行 32 轮，其变体参数为：

```text
delta = 0xB9E1C851
e = (sum >> 3) & 3
MX = ((z >> 2 XOR y << 5) + (y >> 4 XOR z << 3))
     XOR ((sum XOR y) + (K[(p & 3) XOR e] XOR z))
```

每轮加密先更新 `v0` 再更新 `v1`，逆向时必须先减去 `v1` 的 MX，再减去 `v0` 的 MX。还有一个不能照抄标准 XXTEA 的细节：handler 在 `p=0` 时令 `y=z=旧 v1`，在 `p=1` 时令 `y=z=新 v0`。因此逆算 `p=1` 时必须用 `y=z=v0`，恢复 `v1` 后再用 `y=z=v1` 逆算 `p=0`。目标数组为：

```text
2529833208 1707418237 3875301845 2348577753 3088034044
1569396279 1056408561 4141435365 1008452449 1778196976
```

等价解密函数：

```python
def xxtea_variant_decrypt(block, key):
    v0, v1 = block
    delta = 0xB9E1C851
    total = delta * 32 & MASK

    for _ in range(32):
        e = (total >> 3) & 3

        y = z = v0
        p = 1
        mx = (
            ((z >> 2 ^ y << 5) + (y >> 4 ^ z << 3))
            ^ ((total ^ y) + (key[(p & 3) ^ e] ^ z))
        ) & MASK
        v1 = (v1 - mx) & MASK

        y = z = v1
        p = 0
        mx = (
            ((z >> 2 ^ y << 5) + (y >> 4 ^ z << 3))
            ^ ((total ^ y) + (key[(p & 3) ^ e] ^ z))
        ) & MASK
        v0 = (v0 - mx) & MASK
        total = (total - delta) & MASK

    return v0, v1
```

### 整合两段结果

使用小端序将两段解密结果转回字节并拼接：

```python
import struct

key = [0x11223344, 0x55667788, 0x99AABBCC, 0xDDEEFF00]
tea_target = [0x5C6C723E, 0xEF5298D9, 0x62D93D11, 0x098A4E7F]
vm_target = [
    2529833208, 1707418237, 3875301845, 2348577753, 3088034044,
    1569396279, 1056408561, 4141435365, 1008452449, 1778196976,
]

part1 = b"".join(
    struct.pack("<II", *tea_decrypt(tea_target[i:i + 2], key))
    for i in range(0, 4, 2)
)
part2 = b"".join(
    struct.pack("<II", *xxtea_variant_decrypt(vm_target[i:i + 2], key))
    for i in range(0, 10, 2)
)
print(b"miniL{" + part1 + part2)
```

结果为：

```text
miniL{EASY_UPX_AND_TEA___bie_xiao_ni_ye_guo_bu_liao_di_2_guan}
```

运行原程序验证时需避免 `IsDebuggerPresent()` 导致的格式失败，或在离线脚本中直接正向重放两段加密，检查是否精确命中两个目标数组。

## 方法总结

- 核心技巧：把 VEH handler 当作栈式 VM 指令，按注册顺序还原变体 XXTEA，再与明文 TEA 结果拼接。
- 识别信号：全局构造阶段大量调用 `AddVectoredExceptionHandler`，主函数只留一个 `__debugbreak()`，且 handler 反复操作全局寄存器和栈，这是 VEH VM 的明显特征。
- 复用要点：函数名可以故意欺骗，移位量必须从 handler 本体取证；逆转 ARX 算法时必须按加密更新的反顺序处理 word，并在每步限制到 32 bit。
