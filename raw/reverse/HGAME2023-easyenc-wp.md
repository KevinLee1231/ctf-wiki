# easyenc

## 题目简述

程序读取一个长度为 41 的字符串，对每个输入字节先异或 `0x32`，再减去 `86`，然后与内置密文比较。密文在反编译结果中被编译器按 32 位整数分组初始化，需要先按小端序还原成连续字节，再逆向加法和异或。

## 解题过程

在 IDA 中将误识别为 `__int128[3]` 的输入缓冲区改成 `char[48]`，并给变量重命名后，验证循环可以整理为：

```c
if (strlen(input) == 41) {
    for (int i = 0; i < 41; ++i) {
        input[i] = (input[i] ^ 0x32) - 86;
        if (input[i] != ciphertext[i]) {
            fail();
        }
    }
}
```

设明文字节为 $x$，密文字节为 $c$，则：

$$
c = (x \oplus 0x32) - 86
$$

因此逆变换为：

$$
x = (c + 86) \oplus 0x32
$$

反编译器展示的 32 位初始化值如下。用 `& 0xffffffff` 保留补码位模式，再按小端序打包；计算时则把大于等于 `0x80` 的字节解释为有符号 `char`：

```python
words = [
    167640836,
    11596545,
    -1376779008,
    85394951,
    402462699,
    32375274,
    -100290070,
    -1407778552,
    -34995732,
    101123568,
    -7,
]

ciphertext = b"".join(
    (word & 0xFFFFFFFF).to_bytes(4, "little") for word in words
)[:41]

plaintext = "".join(
    chr(((byte if byte < 0x80 else byte - 0x100) + 86) ^ 0x32)
    for byte in ciphertext
)
print(plaintext)
```

运行得到：

```text
hgame{4ddit1on_is_a_rever5ible_0peration}
```

## 方法总结

- 核心技巧：从校验循环推导逐字节逆变换，并正确恢复编译器打包初始化的字节序列。
- 易错点：负整数只是反编译器对同一 32 位位模式的有符号解释；字节运算还要考虑 C 中 `char` 的符号扩展。
- 复用要点：反编译类型不可靠时，应结合 `scanf` 长度、索引方式和比较宽度修正数组类型，再写出明确的正向公式和逆公式。
