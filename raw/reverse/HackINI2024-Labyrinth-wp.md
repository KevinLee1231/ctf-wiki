# HackINI2024 Labyrinth

## 题目简述

题目把 flag 拆成六段并打乱处理，同时混淆了变换函数名。需要先从二进制字符串中按实际存放顺序拼回密文，再识别四个可逆变换，并尝试它们的逆操作顺序。

## 解题过程

在主逻辑附近可以找到六段字符串：

```text
T0p_H4Y3
_!O}nczg
ghvozn{O
0jF_n0_h
pxc_$O3k
n_Wp
```

顺序拼接为：

```text
T0p_H4Y3_!O}nczgghvozn{O0jF_n0_hpxc_$O3kn_Wp
```

对照各函数行为可得到四个逆操作：左旋 12 位、整串反转、逐词反转，以及对英文字母做 Caesar `+5`。当前字符串没有空格，所以“逐词反转”与整串反转效果相同，两个操作组合后会互相抵消。为了避免依赖混淆后的函数名，直接枚举 24 种操作顺序，并为每种排列重新从原密文开始：

```python
from itertools import permutations

ciphertext = "T0p_H4Y3_!O}nczgghvozn{O0jF_n0_hpxc_$O3kn_Wp"

def rotate_left(text):
    return text[12:] + text[:12]

def reverse(text):
    return text[::-1]

def caesar_plus_five(text):
    result = []
    for char in text:
        if "a" <= char <= "z":
            char = chr((ord(char) - ord("a") + 5) % 26 + ord("a"))
        elif "A" <= char <= "Z":
            char = chr((ord(char) - ord("A") + 5) % 26 + ord("A"))
        result.append(char)
    return "".join(result)

operations = [rotate_left, reverse, reverse, caesar_plus_five]

for order in permutations(range(4)):
    candidate = ciphertext
    for index in order:
        candidate = operations[index](candidate)
    if candidate.startswith("shellmates{") and candidate.endswith("}"):
        print(candidate)
        break
```

输出为：

```text
shellmates{T0oK_s0_much_Y0u_M4D3_!T}
```

仓库中的 C 求解器在遍历不同排列时反复修改同一个 `flag` 缓冲区，没有为每个排列恢复初值，因此结果会被前一次变换污染。上面的实现修复了这一问题。

## 方法总结

面对多层可逆变换，先逐个写出严格的逆函数，再枚举很小的操作排列空间，比猜测混淆函数名可靠。枚举时每个候选必须从同一份原始密文开始。两个反转在无空格输入上等价，也解释了为什么多种排列会导向相同明文；最终用完整 flag 前后缀筛选即可。
