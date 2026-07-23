# giedi-composite

## 题目简述

题目实现了一个 NTRU 风格的公钥密码系统。参数为 $N=210$、$q=2003$、$p=3$，运算分别位于

$$
R_q=\mathbb{Z}_q[x]/(x^{210}-1),\qquad
R_p=\mathbb{Z}_p[x]/(x^{210}-1).
$$

私钥 $f$ 和辅助多项式 $g$ 的系数都很小，公钥与密文为

$$
h=p f^{-1}g \pmod q,\qquad
c=bh+m \pmod q,
$$

其中 $b$ 也是小系数多项式，消息 $m$ 被编码为 210 个二进制系数。关键弱点不是 $q$，而是复合维度 $N=210$ 使模多项式可以拆成三个低维因子，从而把一次 210 维 NTRU 格攻击降成三次 70 维攻击。

## 解题过程

### 将环拆成三个低维商环

官方求解脚本使用乘法分解

$$
x^{210}-1=
(x^{70}-x^{35}+1)
(x^{70}+x^{35}+1)
(x^{70}-1).
$$

其中相邻括号表示相乘。记三个因子为 $P_1,P_2,P_3$，分别在
$\mathbb{Z}[x]/(P_i)$ 中投影公钥。这样每个投影只有 70 个系数。

对应的 Sage 代码为：

```sage
Rx.<x> = PolynomialRing(ZZ)

p1 = x^70 - x^35 + 1
p2 = x^70 + x^35 + 1
p3 = x^70 - 1

assert p1 * p2 * p3 == x^210 - 1
```

### 在每个投影中恢复短私钥

由 $hf=pg\pmod q$ 可知，$(f,pg)$ 是标准 NTRU 格中的短向量。对每个 70 维商环，先构造公钥的循环卷积矩阵 $H_i$，再建立 140 维格

$$
L_i=
\begin{pmatrix}
I&H_i\\
0&qI
\end{pmatrix}.
$$

官方脚本对三个格分别执行 BKZ-35，并从约化基中取得短向量的前 70 个坐标作为 $f$ 在对应商环中的候选投影：

```sage
R = Rx.quotient(factor)
deg = factor.degree()

conv = matrix(
    ZZ, deg, deg,
    [list(map(ZZ, (R(pub) * x^i).list())) for i in range(deg)]
)
lat = block_matrix([
    [identity_matrix(ZZ, deg), conv],
    [zero_matrix(ZZ, deg, deg), q * identity_matrix(ZZ, deg)],
])

reduced = lat.BKZ(delta=0.99, block_size=35)
f_part = R(list(reduced.rows()[2][:deg]))
```

BKZ 得到的分量可能带有循环移位或整体符号歧义。官方脚本使用预先计算的多项式 CRT 基把三个投影重新拼回模 $x^{210}-1$ 的多项式，并枚举第二、第三个分量的循环移位。真正的 $f$ 只含少量小系数，因此可用“重建结果的不同系数值不超过五种”筛出候选。

### 解密并验证

对每个候选 $f$ 继续枚举 210 个整体循环移位。NTRU 解密关系为

$$
fc=pbg+fm \pmod q.
$$

将系数中心提升到整数后模 $p$，第一项消失，再乘 $f^{-1}\pmod p$ 即可恢复 $m$：

```sage
v = Rq(f_candidate) * Rq(ct)
v = [coef.lift_centered() for coef in v.list()]

plain_poly = Rp(v) * Rp(f_candidate).inverse()
bits = [coef.lift_centered() for coef in plain_poly.list()]

if all(bit in (0, 1) for bit in bits):
    print(decode_msg(bits))
elif all(bit in (0, -1) for bit in bits):
    print(decode_msg([-bit for bit in bits]))
```

二进制系数检查同时消除了 CRT、符号和旋转产生的错误候选，最终恢复：

```text
UMDCTF{NTRUly_a_n1c3_j0b}
```

## 方法总结

- 核心技巧：利用 $x^{210}-1$ 的低次因子把 NTRU 格投影到三个 70 维商环，分别做 BKZ 后再用多项式 CRT 重建私钥。
- 识别信号：NTRU 参数中的维度 $N$ 为高度合数，且模多项式为 $x^N-1$ 时，应先检查能否分解成显著更低维的因子。
- 复用要点：低维格恢复的投影通常存在符号和循环移位歧义；可结合私钥小系数分布、消息字母表和合法编码结构筛选，而不应把某一条 BKZ 基向量直接当成最终私钥。
