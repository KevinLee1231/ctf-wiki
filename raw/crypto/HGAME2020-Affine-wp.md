# Affine

## 题目简述

题目使用仿射密码，但运算字母表不是常见的 26 个小写字母，而是一张包含大小写字母和数字的 62 字符乱序表。密文保留了 `{`、`}`、`_` 等不在表中的字符，并已知明文以 `hgame{` 开头，可据此求出密钥。

## 解题过程

自定义字母表和密文为：

```python
TABLE = "zxcvbnmasdfghjklqwertyuiop1234567890QWERTYUIOPASDFGHJKLZXCVBNM"
ciphertext = "A8I5z{xr1A_J7ha_vG_TpH410}"
MOD = len(TABLE)  # 62
```

仿射加密满足 $y\equiv Ax+B\pmod{62}$。由已知前缀中的 `h -> A` 与 `g -> 8` 可列出：

$$
\begin{aligned}
A\operatorname{INDEX}(h)+B&\equiv\operatorname{INDEX}(A)\pmod{62},\\
A\operatorname{INDEX}(g)+B&\equiv\operatorname{INDEX}(8)\pmod{62}.
\end{aligned}
$$

两式相减后求模逆，可得 $A=13$、$B=14$，且 $A^{-1}\equiv43\pmod{62}$。完整解密代码如下：

```python
TABLE = "zxcvbnmasdfghjklqwertyuiop1234567890QWERTYUIOPASDFGHJKLZXCVBNM"
ciphertext = "A8I5z{xr1A_J7ha_vG_TpH410}"
modulus = len(TABLE)

index = TABLE.index
A = ((index("A") - index("8"))
     * pow(index("h") - index("g"), -1, modulus)) % modulus
B = (index("A") - A * index("h")) % modulus
A_inv = pow(A, -1, modulus)

plaintext = "".join(
    TABLE[((index(ch) - B) * A_inv) % modulus] if ch in TABLE else ch
    for ch in ciphertext
)
print(A, B, plaintext)
```

输出为：

```text
13 14 hgame{M4th_u5Ed_iN_cRYpt0}
```

## 方法总结

- 核心技巧：利用已知明文建立两个同余式，先求 $A$，再求 $B$ 和 $A^{-1}$。
- 识别信号：字符被固定一一映射，题名或源码提示模运算，且密文保留标点结构。
- 复用要点：必须使用题目给出的字母表顺序和模数；只有 $\gcd(A,M)=1$ 时才存在唯一的解密逆元。
