# easyasm

## 题目简述

附件给出一段 32 位 x86 汇编和一组加密后的字节。核心函数遍历输入字符串，对每个字节原地异或 `0x33`。异或运算满足自反性，因此对密文再次执行相同异或即可恢复明文。

## 解题过程

在 `enc` 函数中，局部变量 `i` 位于 `[ebp-4]`，参数 `Str` 位于 `[ebp+8]`。循环的关键指令为：

```asm
mov     edx, [ebp+Str]
add     edx, [ebp+i]
movsx   eax, byte ptr [edx]
xor     eax, 33h
mov     ecx, [ebp+Str]
add     ecx, [ebp+i]
mov     [ecx], al
```

`edx` 指向 `Str[i]`，函数取出该字节，与 `0x33` 异或后写回原位置，所以等价逻辑为：

```c
for (size_t i = 0; i < strlen(str); ++i) {
    str[i] ^= 0x33;
}
```

题目给出的密文可直接用 Python 还原：

```python
ciphertext = [
    0x5B, 0x54, 0x52, 0x5E, 0x56, 0x48, 0x44, 0x56, 0x5F,
    0x50, 0x03, 0x5E, 0x56, 0x6C, 0x47, 0x03, 0x6C, 0x41,
    0x56, 0x6C, 0x44, 0x5C, 0x41, 0x02, 0x57, 0x12, 0x4E,
]

print(bytes(value ^ 0x33 for value in ciphertext).decode())
```

输出为：

```text
hgame{welc0me_t0_re_wor1d!}
```

## 方法总结

- 核心技巧：识别“读取一个字节—异或常量—写回原地址”的循环。
- 关键性质：`x ^ k ^ k = x`，异或同一常量两次会恢复原值。
- 复用要点：阅读 x86 时先确认栈变量和参数，再追踪地址计算、读写宽度和循环边界，不必逐条解释与数据流无关的函数序言。
