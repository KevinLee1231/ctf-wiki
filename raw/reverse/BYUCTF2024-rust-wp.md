# Rust

## 题目简述

这是一个 stripped Rust flag checker。源码逐字符取输入字节，加 69 后与 24 个整数比较。官方 README 误写为“23 字节含 NUL”；实际 `read_line` 长度检查是 25，其中包括末尾换行，因此可见输入必须恰好 24 字节。

## 解题过程

比较逻辑为：

$$
\text{numbers}[i]=\operatorname{ord}(flag_i)+69.
$$

逐项减去 69 即可逆变换：

```python
numbers = [
    167, 190, 186, 168, 185, 171, 192, 183,
    186, 184, 185, 164, 172, 180, 164, 167,
    183, 183, 183, 183, 183, 183, 183, 194,
]
flag = "".join(chr(x - 69) for x in numbers)
print(flag, len(flag))
```

输出长度为 24：

```text
byuctf{rust_go_brrrrrrr}
```

通过终端提交时再由回车产生第 25 个字节 `\n`，满足 `flag.len() == 25`；之后 `trim()` 去掉换行，正好留下 24 个待比较字符。

## 方法总结

逆向高级语言二进制时，应把标准库噪声收缩为少数业务条件。还要区分缓冲区中的字节长度、trim 后字符数和 C 风格 NUL；本题源码的真实长度语义比官方简述更可靠。
