# assume

## 题目简述

题目模拟 Diffie-Hellman 通信：Alice 发送 $g^a$，Bob 发送 $g^b$，正常共享值为 $g^{ab}$。每个 flag 位置给出 20 组记录，但其中一组会被拦截者替换成随机大写字母和无关的 $g^c$。需要仅凭公开群元素判断哪一个字符来自正常 DH 交换。

模数 $p$ 是 64 bit 素数，$g$ 是模 $p$ 的本原根。这个条件使勒让德符号能够泄漏指数奇偶性。

## 解题过程

对本原根 $g$，有

$$
\left(\frac{g^x}{p}\right)=(-1)^x.
$$

因此可以用 `pow(value, (p - 1) // 2, p)` 判断 $g^x$ 的指数奇偶性：结果为 1 表示偶数，为 $p-1$ 表示奇数。记 $A=g^a$、$B=g^b$、$K=g^{ab}$，则正常记录必须满足

$$
\operatorname{parity}(ab)
=\operatorname{parity}(a)\land\operatorname{parity}(b).
$$

也就是说，只有当 $a$、$b$ 都是奇数时，$g^{ab}$ 才是二次非剩余。拦截者给出的 $g^c$ 与前两个指数无关，每轮约有一半概率违反这一关系。把同一位置的候选字符跨 20 轮累计，真实字符始终不会被淘汰，而随机伪造字符几乎不可能连续通过。

```python
def exponent_is_odd(value: int, p: int) -> bool:
    symbol = pow(value, (p - 1) // 2, p)
    assert symbol in (1, p - 1)
    return symbol == p - 1

def relation_holds(A: int, B: int, K: int, p: int) -> bool:
    expected = exponent_is_odd(A, p) and exponent_is_odd(B, p)
    return exponent_is_odd(K, p) == expected

def recover_position(records, p: int) -> str:
    # records: [(shown_char, A, B, K), ...]
    candidates = {char for char, *_ in records}
    for char, A, B, K in records:
        if not relation_holds(A, B, K, p):
            candidates.discard(char)
    if len(candidates) != 1:
        raise ValueError(f"ambiguous candidates: {candidates}")
    return candidates.pop()
```

按位置恢复并拼接后得到：

```text
tjctf{legendary_legendre0xd5109ab3}
```

## 方法总结

- 题目没有要求求离散对数；真正利用的是群元素的二次剩余类泄漏指数奇偶性。
- 本原根条件很关键。若 $g$ 不是本原根，上述勒让德符号与指数奇偶性的对应关系不能直接套用。
- 单次统计检验会有较高误判率，但 20 次独立记录让错误候选的存活概率指数下降，适合用“持续淘汰”而不是单轮猜测。
