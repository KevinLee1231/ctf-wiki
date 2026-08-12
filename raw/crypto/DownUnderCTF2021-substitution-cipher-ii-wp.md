# DownUnderCTF 2021 - Substitution Cipher II

## 题目简述

题目把 47 个允许字符映射为有限域 $\mathbb F_{47}$ 中的元素，再用一个未知的至多 6 次多项式 $f$ 逐字符替换：

```sage
CHARSET = "DUCTF{}_!?\'" + ascii_lowercase + digits
P.<x> = PolynomialRing(GF(47))
f = P.random_element(6)
cipher_char = CHARSET[f(CHARSET.index(plain_char))]
```

多项式共有 7 个系数，而已知 flag 格式 `DUCTF{...}` 正好提供 7 组明密文对应点：开头六个字符 `DUCTF{` 和末尾的 `}`。

## 解题过程

将字符替换为其在 `CHARSET` 中的索引。设已知点为 $(x_i,y_i)$，用拉格朗日插值恢复唯一的至多 6 次多项式：

$$
f(x)=\sum_{i=0}^{6}y_i
\prod_{j\ne i}\frac{x-x_j}{x_i-x_j}
\quad\text{in }\mathbb F_{47}.
$$

```sage
from string import ascii_lowercase, digits

CHARSET = "DUCTF{}_!?\'" + ascii_lowercase + digits
n = len(CHARSET)
enc = open('../challenge/output.txt').read().strip()

P.<x> = PolynomialRing(GF(n))

known_plain = 'DUCTF{}'
known_cipher = enc[:6] + enc[-1]
points = [(CHARSET.index(p), CHARSET.index(c))
          for p, c in zip(known_plain, known_cipher)]
f = P.lagrange_polynomial(points)
print(f)
```

得到：

$$
f(x)=41x^6+15x^5+40x^4+9x^3+28x^2+27x+1.
$$

有限域上的高次多项式不一定是一一映射。对每个密文索引 $c$，求 $f(x)-c$ 在 $\mathbb F_{47}$ 中的全部根，再对各位置候选做笛卡尔积：

```sage
choices = []
for char in enc:
    value = CHARSET.index(char)
    roots = (f - value).roots()
    choices.append([CHARSET[int(root)] for root, multiplicity in roots])

for candidate in cartesian_product(choices):
    text = ''.join(candidate)
    if text.startswith('DUCTF{') and text.endswith('}'):
        print(text)
```

候选集合很小，结合可读性或逐个提交可以确定：

```text
DUCTF{go0d_0l'_l4gr4ng3}
```

## 方法总结

至多 $d$ 次多项式由 $d+1$ 个不同点唯一确定。题目虽然隐藏了替换多项式，但固定 flag 格式泄露的 7 个字符刚好覆盖 6 次多项式的插值需求。恢复 $f$ 后仍不能假设存在单值逆函数；正确做法是逐密文值求全部有限域根，再利用格式和语言约束组合候选。
