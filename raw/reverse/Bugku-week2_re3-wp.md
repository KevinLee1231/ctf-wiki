# week2_re3

## 题目简述

题目是 64 位 ELF，结合反调试、代码自解密和 RC4。程序在 `.init_array` 中调用 `ptrace(PTRACE_TRACEME)`，非调试状态下才用硬编码 XOR key 解密主变换函数。静态 Ghidra 看到的函数是混淆版，真实逻辑是 RC4。

## 解题过程

### 关键观察

初始化函数逻辑：

```c
FUN_004017b4(0x404a20, "0xdeadbeef", strlen("0xdeadbeef"));
if (ptrace(PTRACE_TRACEME, 0, 1, 0) >= 0) {
    for (i = 1; i < 0x3c0; i++)
        code[0x401215 + i] ^= xor_key[0x4040df + i];
}
```

也就是说，在调试器下 `ptrace` 失败，主函数不会被解密。解密后的真实函数引用 RC4 key `0xdeadbeef`，并对输入做 RC4 加密后与 42 字节密文比较。

### 求解步骤

提取密文 `DAT_004048c0`，用同一 key 解密：

```python
def rc4(data, key):
    s = list(range(256))
    j = 0
    for i in range(256):
        j = (j + s[i] + key[i % len(key)]) & 0xff
        s[i], s[j] = s[j], s[i]
    i = j = 0
    out = bytearray()
    for b in data:
        i = (i + 1) & 0xff
        j = (j + s[i]) & 0xff
        s[i], s[j] = s[j], s[i]
        out.append(b ^ s[(s[i] + s[j]) & 0xff])
    return bytes(out)

expected = bytes([...])  # 42 bytes from DAT_004048c0
print(rc4(expected, b"0xdeadbeef").decode())
```

输出：

```text
flag{25f2d963-27a4-402c-b40c-62d682cf1913}
```

## 方法总结

- 核心技巧：处理反调试保护下的代码自解密，再分析解密后的真实算法。
- 识别信号：`ptrace(PTRACE_TRACEME)`、`.init_array`、静态函数逻辑和密文长度不匹配时，要怀疑代码运行时被修改。
- 复用要点：不要在调试器里直接观察未 patch 版本；先静态复现 XOR 解密，再对真实函数做算法识别。
