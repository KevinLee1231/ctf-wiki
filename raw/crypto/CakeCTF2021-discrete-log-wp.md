# CakeCTF2021 discrete log

## 题目简述

题目对 flag 的每个字节分别计算模幂。设安全素数为 $p$，随机数为 $r$，公开底数为 $g$，第 $i$ 个明文字节为 $m_i$，则密文为

$$
c_i=g^{r m_i}\bmod p.
$$

题名会让人联想到在大素数域上求离散对数，但真正的弱点是所有字符共用了同一个未知乘数 $r$。明文字节又落在很小的可打印字符集合中，因此可以直接比较不同密文之间的指数关系。

## 解题过程

### 消去共同的随机指数

记 $G=g^r\bmod p$，那么 $c_i=G^{m_i}$。以第一个字符 $m_0$ 为基准，有

$$
c_0^{m_i}=G^{m_0m_i}=c_i^{m_0}\pmod p.
$$

等式中已经没有 $g$、$r$，也不要求先恢复 $G$。flag 由可打印 ASCII 组成，因此只要分别枚举 $m_0,m_i\in[0x20,0x7e]$ 即可。错误的 $m_0$ 通常无法为每个位置找到唯一匹配；正确的 $m_0$ 会使整条明文完整出现。

### 枚举可打印字符

仓库官方脚本的核心可以整理为：

```python
import ast

with open("distfiles/output.txt", "r", encoding="utf-8") as f:
    p, g, cs = map(ast.literal_eval, f.read().splitlines())

for m0 in range(0x20, 0x7f):
    plain = [m0]
    for ci in cs[1:]:
        hits = [
            mi for mi in range(0x20, 0x7f)
            if pow(cs[0], mi, p) == pow(ci, m0, p)
        ]
        if len(hits) != 1:
            break
        plain.append(hits[0])
    if len(plain) == len(cs):
        print(bytes(plain))
```

输出为：

```text
CakeCTF{ba37a0f409ef3ec23a6cffbc474a1cef}
```

这里没有真正求解大群上的离散对数，复杂度主要是可打印字符集合上的有限枚举。

## 方法总结

- 核心漏洞是多个明文共享同一个指数因子，交叉幂等式能够把随机数完全消去。
- 遇到形如 $g^{rm_i}$ 的批量密文时，应先比较样本间的代数关系，再考虑通用离散对数算法。
- 小消息空间会把看似困难的群问题降为至多 $95^2$ 量级的字符枚举。
