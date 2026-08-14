# bi0sCTF 2022 - TOO-EC

## 题目简述

服务把 flag 对半切开，分别作为两条不同椭圆曲线上的 ECDSA 私钥。它公开 $N=p_1p_2$、带有三段缺失十六进制数字的 $p_1$、两条曲线的参数和签名接口。每个 nonce 按下式生成：

$$
k=\operatorname{MD5}(d\parallel\operatorname{MD5}(m)),
$$

因此无论曲线阶有多大，$k$ 都只有 128 位。服务最多允许五次操作；一次签名请求会同时返回两条曲线的签名。

解题分为两步：先由部分已知的 $p_1$ 分解 $N$，再分别对两条曲线做小 nonce 格攻击，恢复 flag 的两半。

## 解题过程

### 从残缺素数分解 N

把 $p_1$ 中每一段连续的 `?` 替换为独立变量。若第 $j$ 段从第 $t_j$ 个低位开始、宽度为 $w_j$ 位，则：

$$
P(x_0,\ldots,x_r)=P_{\text{known}}+
\sum_j 2^{t_j}x_j,qquad 0\le x_j<2^{w_j}.
$$

真实的 $P$ 是 $N$ 的大因子，因此在模 $N$ 意义下满足“多项式值与 $N$ 有大公因子”。官方 solver 将该多变量多项式放在 $\mathbb Z_N$ 上，构造 $P^kN^{m-k}$ 的移位格，按各未知块宽度缩放后执行 LLL，并用重建多项式的 Jacobian 迭代求小根：

```python
PR = PolynomialRing(Zmod(N), len(shifts), names)
xs = PR.gens()

P = known_part + sum((1 << bit_off[i]) * xs[i]
                     for i in range(len(xs)))

roots = coppersmith(P, bounds=block_bounds, m=6)
p1 = int(P(*roots))
p2 = N // p1
assert p1 * p2 == N
```

这一步不能把每个 `?` 当成独立十六进制字符暴力枚举；应按连续缺口合并变量，变量界就是对应缺口的位宽。

### 把 128 位 nonce 写成隐藏数问题

对同一条曲线收集三组签名 $(h_i,r_i,s_i)$。ECDSA 等式为：

$$
s_i k_i\equiv h_i+d r_i\pmod q,
$$

等价于：

$$
k_i\equiv h_i s_i^{-1}+d r_i s_i^{-1}\pmod q.
$$

定义：

$$
u_i=h_i s_i^{-1}\bmod q,\qquad
t_i=r_i s_i^{-1}\bmod q.
$$

未知 $k_i<2^{128}$，远小于约 512 位的曲线阶 $q$。可以构造以下有理格；前三行是 $qI$，倒数第二行保存 $t_i$，最后一行保存 $u_i$：

```python
B = 1 << 128
basis = []

for i in range(3):
    row = [0] * 5
    row[i] = q
    basis.append(row)

basis.append([t0, t1, t2, B / q, 0])
basis.append([u0, u1, u2, 0, B])

M = Matrix(QQ, basis)
short = M.LLL()[1]
nonces = short[:3]
```

LLL 找到的短向量前三项对应三个小 nonce（实际使用时需统一短向量的符号）。将任意一个 nonce 代回即可得到私钥：

```python
def recover_private(signatures, q):
    # construct_lattice 与上面的矩阵一致
    v = construct_lattice(signatures, q).LLL()[1]
    ks = [Integer(x) % q for x in v[:len(signatures)]]

    ds = [
        (s * k - h) * inverse_mod(r, q) % q
        for (h, r, s), k in zip(signatures, ks)
    ]
    assert all(x == ds[0] for x in ds)
    return ds[0]
```

对曲线 1、曲线 2 分别执行一次，按大端字节拼接两个私钥：

```python
left = long_to_bytes(int(recover_private(curve1_sigs, q1)))
right = long_to_bytes(int(recover_private(curve2_sigs, q2)))
print(left + right)
```

得到：

```text
bi0sctf{ecdsa_on_2_curves_is_not_secure_at_all_33e4a456fd}
```

## 方法总结

“同时使用两条曲线”并不会自动提升安全性：两条曲线都复用了同一种只有 128 位输出的 nonce 生成方式，而私钥又只是 flag 的两半，所以可以独立攻破后拼接。题目的另一个障碍是曲线素数被部分遮挡；部分已知因子的多变量小根恢复与小 nonce 的隐藏数攻击是两次不同的格应用，前者恢复模数，后者恢复私钥。验证时应分别检查 $p_1p_2=N$、每条曲线三次签名导出的 $d$ 一致，以及最终字节拼接顺序。
