# UMDCTF 2025 - deobfuscation

## 题目简述

附件 `flag` 是 64 位、静态链接、已去符号的 ELF。程序读取一行长度恰好为 52 的
输入，逐字节与内置 `key_bytes` 异或，再和 `encrypted_flag` 比较。所谓
“deobfuscation”并没有复杂加壳或自修改代码，关键只是从汇编中识别两组常量和 XOR
校验方向。

## 解题过程

核心循环可还原为：

```asm
mov al, [input + rcx]
cmp al, 10
je  check_flag_len
xor al, [key_bytes + rcx]
mov [decoded + rcx], al
inc rcx
jmp decode_loop
```

随后逐字节比较：

```asm
mov al, [encrypted_flag + rcx]
mov bl, [decoded + rcx]
cmp al, bl
jne wrong
```

因此正确输入满足：

```text
input[i] ^ key_bytes[i] = encrypted_flag[i]
```

XOR 是自身的逆运算，直接得到：

```text
input[i] = encrypted_flag[i] ^ key_bytes[i]
```

从 `.data` 中提取两组各 52 字节的数组后，最小恢复脚本如下：

```python
encrypted_flag = [
    0x20, 0x22, 0x20, 0x26, 0x35, 0x37, 0x14, 0x07,
    # 后续字节按附件中的数组继续
]
key_bytes = [
    0x75, 0x6f, 0x64, 0x65, 0x61, 0x71, 0x6f, 0x75,
    # 后续字节按附件中的数组继续
]

answer = bytes(a ^ b for a, b in zip(encrypted_flag, key_bytes))
print(answer.decode())
```

完整数组异或后输出：

```text
UMDCTF{r3v3R$E-i$_Th3_#B3ST#_4nT!-M@lW@r3_t3chN!Qu3}
```

输入该 52 字节字符串并换行，程序会打印 `Correct!`。

## 方法总结

本题强调先读数据流再判断“加密”强度。比较的是
`input XOR key` 与密文，而不是输入与某个不可逆摘要，所以两组常量足以直接恢复
明文。对小型 stripped ELF，可先从 `strings` 找提示语和数据区，再围绕引用位置阅读
循环；不必因为没有符号就立即进行动态爆破。
