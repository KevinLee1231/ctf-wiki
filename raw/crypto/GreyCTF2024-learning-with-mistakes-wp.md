# learning-with-mistakes

## 题目简述

题目模仿 LWE，但运算环选为 $\operatorname{GF}(2^{32})$，秘密向量是 500 个二进制位。每个已知明文 nibble 产生一个样本，噪声只占域元素的低 28 个系数。由于域加法就是按位异或，高 4 个系数完全无噪声，问题退化为普通的 $\operatorname{GF}(2)$ 线性方程组。

## 解题过程

单个样本满足

$$
b=\langle a,s\rangle+mX^{28}+e,
$$

其中 $s_i\in\{0,1\}$，消息 $m$ 只有 4 位，噪声 $e$ 由 28 位随机数映射而来，因此其多项式次数低于 28。观察 $X^{28}$ 至 $X^{31}$ 的系数，噪声项消失：

$$
\operatorname{high}_4(b)
=\sum_{i=0}^{499}s_i\operatorname{high}_4(a_i)
\oplus m.
$$

题目同时给出用于生成样本的完整已知消息。把消息拆成 nibble，对每个密文样本提取所有 $a_i$ 和 $b$ 的最高 4 位，即可得到四行二元方程：

```python
def equations_from_sample(sample, nibble):
    a, b = sample
    A = matrix(GF(2), [
        list(map(int, format(value, "032b")[:4]))
        for value in a
    ]).T
    high_b = vector(GF(2), map(int, format(b, "032b")[:4]))
    high_m = vector(GF(2), map(int, format(nibble, "04b")))
    return A, high_b - high_m
```

纵向拼接所有样本得到 $As=v$，在 $\operatorname{GF}(2)$ 上求一个特解并枚举右核中的少量候选。题目对秘密位串计算 SHA-512，再与 flag 异或，因此用每个候选重建相同密钥流并检查 `grey{` 前缀：

```python
for candidate in all_solutions(A, v):
    key_int = int("".join(map(str, candidate)), 2)
    stream = sha512(long_to_bytes(key_int)).digest()
    plaintext = xor(bytes.fromhex(flag_xored), stream)
    if plaintext.startswith(b"grey{"):
        print(plaintext)
```

最终恢复：

```text
grey{I'm_flyin_soon_I'm-_rushing-this-challenge-rn-ajsdadsdasks}
```

## 方法总结

把模数环替换为扩域并不会自动增强 LWE。这里秘密仅取二进制值，乘法退化为“选或不选”公开域元素；更致命的是噪声被限制在固定低位子空间，投影到互补的高位子空间后可被完全消去。分析带结构噪声的 LWE 变体时，应寻找能消灭噪声的线性投影，而不是直接套用格攻击。
