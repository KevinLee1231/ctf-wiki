# LLLCG

## 题目简述

题目把 flag 转成整数作为 LCG 的乘数 $a$，随机初始状态小于约 $2^{360}$，每次再加入一个至多模数大小的随机扰动。代码原本想在模 $n$ 下更新状态，但括号缺失导致取模只作用在随机数上，状态本身不断增大。目标是从 40 个连续输出中恢复 $a$。

## 解题过程

关键代码为：

```python
from Crypto.Util.number import bytes_to_long
from random import randint
from sage.all import next_prime


class LCG:
    def __init__(self, flag):
        self.n = next_prime(2**360)
        self.a = bytes_to_long(flag)
        self.seed = randint(1, self.n - 1)

    def next(self):
        self.seed = self.seed * self.a + randint(-2**340, 2**340) % self.n
        return self.seed
```

Python 的 `%` 优先级高于 `+`，实际递推不是预期的

$$
s_{i+1}=(a s_i+r_i)\bmod n,
$$

而是

$$
s_{i+1}=a s_i+R_i,
\qquad 0\le R_i<n,
$$

其中 $R_i=r_i\bmod n$。第一次更新后，$s_1$ 已经远大于 $n$；对后续相邻项有：

$$
s_{i+1}=a s_i+R_i,
\qquad 0\le R_i<n<s_i.
$$

因此余项严格小于除数，直接整除便有：

$$
\left\lfloor\frac{s_{i+1}}{s_i}\right\rfloor=a.
$$

这不是近似估计，而是在第一项之后就能精确恢复乘数。读取附件中的输出列表后逐对尝试即可：

```python
from ast import literal_eval
from Crypto.Util.number import long_to_bytes

with open("output.txt", "r", encoding="utf-8") as stream:
    outputs = literal_eval(stream.read())

for previous, current in zip(outputs, outputs[1:]):
    candidate = current // previous
    decoded = long_to_bytes(candidate)
    if decoded.startswith(b"hgame{"):
        print(decoded)
        break
```

输出为：

```text
b'hgame{W0w_you_know_the_hidden_number_problem}'
```

官方 PDF 说明了缺括号造成的非预期解，但未保留输出数据和最终结果；flag 由参赛者的 [HGAME 2023 Week 4 题解](https://lazzzaro.github.io/2023/02/06/match-HGAME-2023-Week-4/index.html) 交叉核验。

## 方法总结

密码题中的运算优先级错误会彻底改变数学模型。看到“LCG 加噪声”时，应先按语言真实语义重写递推式，而不是直接套 LLL。这里状态没有模约减，迅速增长到远大于噪声上界，相邻输出的整数商就直接泄露了作为 flag 的乘数。
