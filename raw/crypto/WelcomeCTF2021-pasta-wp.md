# Pasta

## 题目简述

WelcomeCTF2021 的 Pasta 给出密文：

```text
terlungf{J3YP0Z3_G0_PELCG0TE4CUL}
```

题目中的 Pasta、Rome 等提示指向凯撒密码，且密文中的括号、数字与下划线都保持不变。

## 解题过程

已知比赛 flag 前缀是 `greyhats{`。比较 `terlungf{` 与目标前缀可发现每个字母相差 13 位，因此使用 ROT13。ROT13 是凯撒移位 13 的特例，对英文字母执行两次会回到原文。

可以直接用 Python 标准库验证：

```python
import codecs

ciphertext = "terlungf{J3YP0Z3_G0_PELCG0TE4CUL}"
print(codecs.decode(ciphertext, "rot_13"))
```

输出为：

```text
greyhats{W3LC0M3_T0_CRYPT0GR4PHY}
```

数字、下划线和花括号不属于字母表，解码时原样保留。

## 方法总结

这道题的关键是利用题面语义和已知 flag 前缀确定移位量，而不是盲试所有编码。对凯撒密码而言，即使没有提示，也只需枚举 26 个移位并检查可读性；ROT13 的自反性质使验证尤其简单。
