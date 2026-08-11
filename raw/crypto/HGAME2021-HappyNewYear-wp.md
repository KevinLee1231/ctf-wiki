# HappyNewYear!!

## 题目简述

题目给出 7 组 RSA 公钥与密文，所有公钥指数均为 $e=3$。其中有若干收件人收到完全相同的明文，而且加密前没有随机填充；同一明文在至少 3 个两两互素模数下加密时，可以用中国剩余定理恢复整数 $m^3$，再开精确三次方根。这是低指数广播攻击。数据中存在两组重复明文，flag 分藏在两段结果中。

## 解题过程

对同一个明文整数 $m$，三组密文满足

$$
c_i\equiv m^3\pmod{n_i},\qquad i\in\{1,2,3\}.
$$

若 $n_1,n_2,n_3$ 两两互素，中国剩余定理可以在模 $N=n_1n_2n_3$ 下唯一恢复 $x\equiv m^3\pmod N$。只要明文足够短，就有 $m^3<N$，因此 CRT 得到的 $x$ 就是普通整数意义下的 $m^3$，对其开精确立方根即可取得 $m$。

问题是不知道 7 组数据中哪 3 组加密了同一消息，所以枚举全部 $\binom{7}{3}=35$ 个三元组合。错误组合得到的 CRT 结果几乎不可能是完全立方数，`iroot(x, 3)` 的精确标志可直接用作筛选条件。

官方附件 `output` 每组占 5 行，前三行依次含 `n`、`e`、`c`。下面保留这一格式并补全异常处理：

```python
from functools import reduce
from itertools import combinations
import operator

import gmpy2
from Crypto.Util.number import long_to_bytes


def crt(moduli, residues):
    modulus_product = reduce(operator.mul, moduli, 1)
    result = 0
    for modulus, residue in zip(moduli, residues):
        partial = modulus_product // modulus
        inverse = pow(partial, -1, modulus)
        result += residue * inverse * partial
    return result % modulus_product


with open("output", "r", encoding="utf-8") as stream:
    lines = stream.read().splitlines()

parameters = []
for index in range(0, len(lines), 5):
    n = int(lines[index].split()[-1])
    e = int(lines[index + 1].split()[-1])
    c = int(lines[index + 2].split()[-1])
    if e != 3:
        raise ValueError(f"unexpected public exponent: {e}")
    parameters.append((n, c))

parts = []
for group in combinations(parameters, 3):
    moduli = [item[0] for item in group]
    residues = [item[1] for item in group]
    try:
        cube = crt(moduli, residues)
    except ValueError:
        continue

    root, exact = gmpy2.iroot(cube, 3)
    if exact:
        plaintext = long_to_bytes(int(root))
        if plaintext not in parts:
            parts.append(plaintext)
            print(plaintext)
```

运行后会出现两段不同明文，按 flag 语义拼接即可。官方 PDF 没有保留 7 组实例参数，也没有记录两段运行输出，因此这里不猜测最终 flag；保留下来的信息足以完整复现算法，但实际结果仍需原始 `output`。

## 方法总结

低指数本身不必然破坏 RSA，真正的问题是“小指数 + 相同明文 + 多个互素模数 + 无随机填充”。广播攻击先用 CRT 消去模约束，再把模方程变为普通整数开根。面对混杂的多组消息时，可以枚举满足阈值数量的组合，以“CRT 结果是否为精确 $e$ 次幂”自动识别属于同一明文的组；本题还要注意会得到两段有效结果，而不是找到第一段就停止。
