# Whose Base Is It Anyway

## 题目简述

题目给出一段几乎只由 `A`、`B` 组成的长文本。源码没有使用标准 Base64，而是把字节流依次按 6、5、4、3、2、1 位切片，再用 Base64 字母表索引切片值；解题时需要按相反过程逐层拼回字节。

## 解题过程

编码器的 `basenencode(data, n)` 从每个字节连续取 $n$ 位，并将数值作为下标映射到：

```text
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/
```

生成器按 `n = 6, 5, 4, 3, 2, 1` 连续编码，所以最终外层的每个符号只承载 1 位，才会只出现字母表索引 0、1，即 `A` 和 `B`。解码时按 `n = 1` 到 `6` 逐层读取符号索引，并把每个索引的低 $n$ 位连续拼成 8 位字节：

```python
for n in range(1, 7):
    ciphertext = basendecode(ciphertext, n)
print(ciphertext)
```

逐层剥离后得到：

```text
UMDCTF-{1t_w@s_my_b@535}
```

## 方法总结

“Base-N”不一定是标准库里的 Base16、Base32 或 Base64。本题的 `n` 表示每个输出符号承载的位数，核心是位流重组。逆向自定义编码时应先记录切片宽度、位序、字符表和尾部剩余位的处理，再严格反转层次顺序。
