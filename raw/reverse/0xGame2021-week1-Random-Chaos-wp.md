# week1Random Chaos

## 题目简述

程序用固定种子 `0x2021` 初始化 Windows/MSVCRT 的 `rand()`，并把每个输入字符与 `rand() & 0xff` 异或后和常量数组比较。固定种子会产生完全确定的伪随机序列，因此可以重放密钥流恢复 flag。

## 解题过程

MSVCRT 的状态更新和返回值可写为：

$$
state_{n+1}=(214013\times state_n+2531011)\bmod 2^{32}
$$

$$
rand_n=(state_{n+1}\gg16)\mathbin{\&}0x7fff
$$

使用相同算法逐字节异或。显式实现 PRNG 可以避免在 Linux 上调用不同实现的 `rand()` 而得到错误序列：

```python
enc = [
    0x22, 0xca, 0x07, 0x19, 0xf8, 0xfb, 0x28, 0x9d,
    0x1e, 0x80, 0xac, 0xc9, 0x60, 0x46, 0x18, 0x21,
    0xdf, 0x95, 0xd5, 0x70, 0xc5, 0x19, 0xea, 0xb0,
    0x9c, 0x83, 0x11, 0x4a, 0x93, 0xc7, 0x91, 0xf6,
    0x14, 0x71, 0x2f, 0x22, 0x14, 0xbf, 0x58, 0x76,
]

state = 0x2021
plain = bytearray()

for value in enc:
    state = (214013 * state + 2531011) & 0xffffffff
    rand_value = (state >> 16) & 0x7fff
    plain.append(value ^ (rand_value & 0xff))

print(plain.decode())
```

输出为：

```text
0xGame{d6ca93397ecb4d4e83792a7100737932}
```

## 方法总结

本题利用的是固定种子的可预测性，不是破解真正随机数。复现 C/C++ `rand()` 时必须确认运行库实现；相同种子在 MSVCRT 与 glibc 下并不保证产生相同序列，因此正文直接保留了题目二进制对应的状态更新公式。
