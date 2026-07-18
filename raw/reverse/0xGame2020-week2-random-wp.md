# week2random?

## 题目简述

附件是 32 位 Windows PE。程序使用固定种子 `0x53df60` 初始化 `srand`，再把内置数组逐字节与 `rand()` 返回值异或。`rand()` 是确定性伪随机数生成器，相同实现和种子会产生相同序列，因此可以离线重放。

需要注意，C 标准没有规定 `rand()` 的具体算法。该 PE 由 MinGW 构建并导入 Windows CRT 的 `rand`，不能直接拿 glibc 下编译的同一段 C 代码期待相同结果。

## 解题过程

Windows CRT 的状态更新和输出为：

$$
s_{i+1}=(214013s_i+2531011)\bmod2^{32}
$$

$$
r_i=(s_{i+1}\gg16)\mathbin{\&}\mathtt{0x7fff}
$$

数组元素是 `unsigned char`，赋值时最终只保留 `rand()` 的低 8 位：

```python
cipher = [
    0x90, 0x21, 0xE5, 0x1D, 0xB0, 0xDC, 0x21, 0x3B,
    0xD5, 0xD0, 0x95, 0xBC, 0x04, 0xB5, 0xB8, 0x3D,
    0xB1, 0xDA, 0xCE, 0xFA, 0x06, 0x5C, 0x21, 0xA4,
    0xF2, 0x8A, 0x78, 0xDC,
]

state = 0x53DF60
plain = bytearray()

for value in cipher:
    state = (state * 214013 + 2531011) & 0xFFFFFFFF
    random_value = (state >> 16) & 0x7FFF
    plain.append(value ^ (random_value & 0xFF))

print(plain.decode())
```

输出为：

```text
0xgame{random_is_not_random}
```

## 方法总结

- 核心技巧：确定 PRNG 实现和固定种子，重放同一随机序列完成异或逆运算。
- 识别信号：`srand` 使用常量种子，随后用 `rand()` 参与可逆校验或加密。
- 复用要点：`rand()` 算法依赖运行库；先确认目标是 MSVCRT、glibc 还是其他实现，并匹配整数宽度与截断行为。
