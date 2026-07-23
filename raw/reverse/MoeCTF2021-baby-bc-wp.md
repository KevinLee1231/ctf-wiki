# baby-bc

## 题目简述

附件是 LLVM IR 文件 `baby_bc.ll`。程序要求输入长度为 40 的字符串，先用内置密钥执行 RC4，再经过自定义的 `HSencode` 编码，最后与常量字符串比较。

这题的关键不在逐行阅读 LLVM 指令，而在于识别出两层可逆变换，并准确还原目标串中的反斜杠和右方括号。

## 解题过程

可以先把 LLVM IR 编译为便于分析的本地程序：

```bash
llvm-as baby_bc.ll -o baby_bc.bc
llc baby_bc.bc -o baby_bc.s
clang baby_bc.s -o baby_bc
```

结合 IR 中的全局常量和循环结构，可以整理出程序的核心逻辑：

1. RC4 密钥共有 14 字节，前三字节是 `11 45 14`，后面是 ASCII 字符串 `avalon,yyds`。
2. RC4 的 KSA 和 PRGA 都是标准实现。
3. `HSencode` 每次把 3 字节拆成 4 个 6 位整数，但没有查 Base64 字符表，而是直接给每个整数加上 61，末尾仍用 `=` 补齐。
4. 编码结果必须等于：

```text
@BdxRTbRBbjIVf`PEyqe^\^\|cc|JRubaGLytHeRI@jgNegHU[Myy]==
```

目标串在 LLVM IR 中的反斜杠写成 `\5C`。此外，末尾是 `[Myy]==`，其中的 `]` 不能漏掉；漏掉后字符串只有 55 字节，无法对应 40 字节输入应产生的 56 字节编码。

逆向时先对每个非填充字符减去 61，再按 Base64 的位拼接方式恢复 RC4 密文，最后使用同一密钥运行 RC4 即可：

```python
target = r"@BdxRTbRBbjIVf`PEyqe^\^\|cc|JRubaGLytHeRI@jgNegHU[Myy]=="


def hsdecode(data: str) -> bytes:
    values = bytes(ord(ch) - 61 for ch in data if ch != "=")
    result = bytearray()

    for offset in range(0, len(values), 4):
        group = values[offset:offset + 4]
        if len(group) >= 2:
            result.append((group[0] << 2) | (group[1] >> 4))
        if len(group) >= 3:
            result.append(((group[1] & 0x0f) << 4) | (group[2] >> 2))
        if len(group) >= 4:
            result.append(((group[2] & 0x03) << 6) | group[3])

    return bytes(result)


def rc4(data: bytes, key: bytes) -> bytes:
    box = list(range(256))
    j = 0

    for i in range(256):
        j = (j + box[i] + key[i % len(key)]) & 0xff
        box[i], box[j] = box[j], box[i]

    i = 0
    j = 0
    result = bytearray()

    for value in data:
        i = (i + 1) & 0xff
        j = (j + box[i]) & 0xff
        box[i], box[j] = box[j], box[i]
        stream_byte = box[(box[i] + box[j]) & 0xff]
        result.append(value ^ stream_byte)

    return bytes(result)


key = bytes([0x11, 0x45, 0x14]) + b"avalon,yyds"
ciphertext = hsdecode(target)
flag = rc4(ciphertext, key)

assert len(ciphertext) == 40
print(flag.decode())
```

运行后得到：

```text
moectf{Y0u_Kn0w_1lVm_ir_c0d3_A_l0t_!1!1}
```

## 方法总结

面对 LLVM IR，先从字符串常量、比较位置和循环边界恢复高层数据流，通常比顺着 SSA 变量逐条追踪更高效。本题还需要特别注意表示层：`\5C` 是 LLVM 对反斜杠字节的转义，而目标串中的任意一个字符缺失都会破坏编码长度关系。识别出“RC4 加密后再做类 Base64 编码”后，两层操作都可以直接逆转。
