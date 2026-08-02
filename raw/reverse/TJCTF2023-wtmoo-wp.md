# wtmoo

## 题目简述

程序把输入中的字母和数字按四段规则平移，再与硬编码字符串比较。花括号保持不变；其他字符会被拒绝。这是一个逐字符可逆替换，没有跨字符状态。

## 解题过程

正向规则为：

```text
a-z  减 60
A-Z  加 32
0-4  加 43
5-9  减 21
{ }  不变
```

对硬编码密文按输出值所在区间应用相反位移：

```python
encrypted = r'''8.'8*{;8m33[o[3[3[%")#*\}'''
plain = []

for char in encrypted:
    value = ord(char)
    if ord("a") - 60 <= value <= ord("z") - 60:
        value += 60
    elif ord("A") + 32 <= value <= ord("Z") + 32:
        value -= 32
    elif ord("0") + 43 <= value <= ord("4") + 43:
        value -= 43
    elif ord("5") - 21 <= value <= ord("9") - 21:
        value += 21
    plain.append(chr(value))

print("".join(plain))
```

输出为：

```text
tjctf{wtMoo0O0o0o0a7e8f1}
```

将结果交给原程序后，变换结果与内置密文完全相同。

## 方法总结

- 分段替换应先把正向输入区间映射到密文区间，再按这些密文区间选择逆操作。
- 规则逐字符独立时无需暴力枚举；直接逆变换并代回比较即可。
- C 字符串中的反斜杠需要区分转义和真实字符，复制密文时应使用原始字符串或字节表示。
