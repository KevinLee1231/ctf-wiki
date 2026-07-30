# SU_Poly

## 题目简述

题目在 SageMath 10.5 中建立

```python
PR.<x> = PolynomialRing(Zmod(2^128 - 2))
SUPOLY = PR.random_element(10)
```

随后生成 $21333=\operatorname{bytes\_to\_long}(\texttt{b"SU"})$ 个随机十次多项式 $f$，泄露每个 $f\cdot SUPOLY$ 在 $0,\ldots,9$ 处函数值的低 8 位。选手必须在 10 秒内提交 `md5(str(SUPOLY.list()))`。

题面看似隐藏数问题，实际突破口是两个实现细节：模数 $2^{128}-2$ 可降到模 $2$；Sage 多项式系数最终由 Python 的 MT19937 生成。泄露量足以线性恢复随机数生成器状态。

## 解题过程

### 把多项式泄露降到模 2

只看每组 `gift` 的第一个值，即 $x=0$：

$$
(f\cdot SUPOLY)(0)=f(0)\,SUPOLY(0)\pmod{2^{128}-2}.
$$

因为模数为偶数，再取最低位可得

$$
\operatorname{LSB}((f\cdot SUPOLY)(0))
=\operatorname{LSB}(f(0))\operatorname{LSB}(SUPOLY(0)).
$$

- 若 `SUPOLY` 常数项为偶数，所有泄露位均为 $0$，断开并重新连接；
- 若其常数项为奇数，每组第一个函数值的最低位就是随机多项式 $f$ 的常数项最低位。

```python
known = [row[0] & 1 for row in gift]
if 1 not in known:
    reconnect()
```

### 对齐 Sage 的随机系数生成顺序

Sage 10.5 的 `PolynomialRing.random_element(10)` 逐个调用底层环的 `random_element` 产生 11 个系数，并按最高次项到常数项的顺序填充。对 `Zmod(2^128-2)` 而言，这近似对应 Python `randrange(0, 2^128-2)`，除去概率极低的拒绝采样，等价于连续消费 128 个 MT19937 输出位。

因此连接建立后的随机调用顺序为：

1. 先消费 11 个 128-bit 值生成 `SUPOLY`；
2. 每个随机 $f$ 再消费 11 个 128-bit 值；
3. 每组可观测位是该组最后一个值，也就是 $f(0)$ 的最低位。

这一实现关系已经足够复现攻击，不必依赖外链；Sage 源码中的关键行为就是“系数逐个委托给基环随机采样，并从高次向低次填充”。

### 预计算 MT19937 线性逆映射

MT19937 的内部状态可写成 $624\times32=19968$ 位，但线性状态空间的有效维数为 19937。对每个状态基向量模拟与服务端相同的随机调用顺序，便可得到“初始状态位 $\rightarrow$ 21333 个泄露位”的 $\mathrm{GF}(2)$ 线性映射：

```python
def construct_row(rng):
    rng.getrandbits(128 * 11)  # SUPOLY
    row = []
    for _ in range(bytes_to_long(b"SU")):
        rng.getrandbits(128 * 10)
        row.append(rng.getrandbits(128) & 1)
    return row

rows = []
for i in range(19968):
    state_bits = "0" * i + "1" + "0" * (19967 - i)
    state = [int(state_bits[32*j:32*j+32], 2) for j in range(624)]
    rng.setstate((3, tuple(state + [624]), None))
    rows.append(construct_row(rng))

L = Matrix(GF(2), rows).T
K = L[:19937, [0] + list(range(32, 19968))]
K_inv = K.inverse()
```

矩阵逆必须离线预计算；远端 10 秒窗口内只做一次矩阵—向量乘法：

```python
leak = vector(GF(2), known[:19937])
reduced_state = (K_inv * leak).list()
state_bits = [reduced_state[0]] + [0] * 31 + reduced_state[1:]
```

把 19968 位重新分组为 624 个 32-bit 状态字并调用 `setstate`，随后重放最开始的 11 次 128-bit 取样。由于 Sage 从高次项向常数项填充，预测结果需逆序后才等于 `SUPOLY.list()`：

```python
rng.setstate((3, tuple(state_words + [624]), None))
supoly = [rng.getrandbits(128) for _ in range(11)][::-1]
answer = md5(str(supoly).encode()).hexdigest()
```

提交 `answer` 即可得到源码中记录的 flag：

```text
SUCTF{bytes_to_long_of_SU_seems_bigger_than_19937_XD_try_random_matrix(GF(2),20000,20000)_and_see_its_rank!}
```

## 方法总结

- 核心技巧：将复合模数上的多项式乘积降到模 $2$，把常数项最低位泄露转化为 MT19937 的线性观测，再预计算逆矩阵恢复状态。
- 识别信号：大量随机对象由同一个非密码学 PRNG 连续生成，而且输出泄露单比特线性函数时，应比较泄露数量与 PRNG 的线性状态维数。
- 复用要点：攻击不仅取决于算法，还取决于库版本、系数填充顺序和 `randrange` 的消费方式；这些调用顺序必须按部署环境复现。限时题应把矩阵构造和求逆放到离线阶段。
