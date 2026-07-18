# week1supperAffine

## 题目简述

题目在长度为 62 的字符表 `ascii_letters + digits` 上连续执行三次仿射变换。花括号和连字符不在字符表内，会原样保留；其余字符先转换为下标，再对 62 取模。

三层变换可以合并为一层：

$$
y \equiv A x + B \pmod{62}
$$

其中

$$
A \equiv a_2a_1a_0 \pmod{62}
$$

$$
B \equiv a_2a_1b_0+a_2b_1+b_2 \pmod{62}
$$

## 解题过程

flag 已知以 `0xGame{` 开头，密文以 `t6b7Tn{` 开头，因此可以用明密文对 `0xGa -> t6b7` 枚举合并后的 $A,B$。唯一满足条件的参数为：

$$
A=35,\qquad B=59
$$

且 $35^{-1}\equiv39\pmod{62}$，所以解密公式是：

$$
x \equiv A^{-1}(y-B) \pmod{62}
$$

完整脚本如下：

```python
from string import ascii_letters, digits

table = ascii_letters + digits
modulus = len(table)
cipher = "t6b7Tn{2GByBZBB-aan2-JRWn-GnZB-Jyf7a722ffnZ}"

pairs = []
for a in range(modulus):
    for b in range(modulus):
        if all(
            (a * table.index(plain) + b) % modulus == table.index(enc)
            for plain, enc in zip("0xGa", cipher[:4])
        ):
            pairs.append((a, b))

assert pairs == [(35, 59)]
A, B = pairs[0]
A_inv = pow(A, -1, modulus)

flag = ""
for char in cipher:
    if char not in table:
        flag += char
        continue
    index = A_inv * (table.index(char) - B) % modulus
    flag += table[index]

print(flag)
```

输出为：

```text
0xGame{1b292822-33e1-46fe-be82-49ca3a11cce8}
```

## 方法总结

- 核心技巧：先合并多层仿射变换，再用已知明文恢复等效参数。
- 识别信号：每层都是一次式，且运算始终在同一个模数下进行。
- 复用要点：只有 $\gcd(A,62)=1$ 时乘法逆元存在；字符表的顺序也是密钥的一部分，不能换成自定义顺序。
