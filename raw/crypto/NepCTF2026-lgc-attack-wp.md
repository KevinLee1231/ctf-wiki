# NepCTF2026 LGC Attack Writeup

## 题目简述

题目组合了两个泄漏：

1. RSA 素因子 $p$ 的高位已知，只隐藏低 502 位；
2. flag 被补零到 64 字节后作为 LCG 初始状态，连续输出六个状态的高位，每次隐藏低 256 位。

需要先用 Coppersmith 恢复 $p$，再利用截断 LCG 的线性关系构造格，通过 LLL 恢复初始状态低位。

## 解题过程

### Coppersmith 恢复 RSA 素因子

已知：

$$
p=p_{\text{high}}2^{502}+x,\qquad 0\le x<2^{502},
$$

且 $p\mid N$。理论上可对：

$$
f(x)=p_{\text{high}}2^{502}+x
$$

直接做 `small_roots`，但 502 位接近该实例和参数下的实际求解边界。将未知量拆成高 6 位与低 496 位：

$$
x=t2^{496}+y,\qquad 0\le t<2^6,\quad 0\le y<2^{496}.
$$

枚举 $t$，对每个候选求小根：

```python
PR.<y> = PolynomialRing(Zmod(N))

for t in range(1 << 6):
    known = ((p_high << 6) + t) << 496
    f = y + known
    roots = f.small_roots(
        X=2^496, beta=0.4, epsilon=0.01
    )
    if roots:
        p = known + int(roots[0])
        if N % p == 0:
            break
```

这样把 Coppersmith 的未知根从 502 位降到 496 位，只需额外枚举 64 个候选。

### 建立截断 LCG 关系

状态更新为：

$$
s_{i+1}=a(s_i-c)\pmod p.
$$

令 $K=2^{256}$，公开输出为 $o_i=\lfloor s_i/K\rfloor$，隐藏低位为：

$$
s_i=o_iK+\ell_i,\qquad 0\le\ell_i<K.
$$

展开递推：

$$
s_i=a^i s_0+C_i\pmod p,
$$

其中：

$$
C_i=-c\sum_{j=1}^{i}a^j\pmod p.
$$

代入高低位分解，得到：

$$
a^i\ell_0-\ell_i+V_i\equiv0\pmod p,
$$

$$
V_i=a^i o_0K+C_i-o_iK\pmod p.
$$

未知的 $\ell_0,\ldots,\ell_5$ 均小于 $K$。构造 $7\times7$ 整数格：

```python
K = 2^256
V = []

for i in range(1, 6):
    C_i = -sum(pow(a, j, p) * c for j in range(1, i + 1))
    C_i %= p
    V_i = (
        pow(a, i, p) * outputs[0] * K
        + C_i - outputs[i] * K
    ) % p
    V.append(V_i)

M = Matrix(ZZ, 7, 7)
M[0, 0] = 1
for i in range(1, 6):
    M[0, i] = pow(a, i, p)
    M[i, i] = p
    M[6, i] = -V[i - 1]
M[6, 6] = K
```

执行 `M.LLL()`，寻找最后一维为 `K` 或 `-K` 的短向量；按符号修正后，其第一维即 $\ell_0$。恢复：

```python
seed = outputs[0] * K + l0
flag = long_to_bytes(int(seed)).rstrip(b"\x00")
print(flag.decode())
```

得到：

```text
Nepctf{C0pp3rsm1th_m33ts_LLL_in_Latt1c3_w0rld!}
```

该题官方输出中的 `Nepctf` 大小写与常见前缀不同，应按实际解密字节保留。

## 方法总结

本题展示了两种“接近边界但可组合”的泄漏。Coppersmith 直接处理 502 位低位不稳定时，可以枚举少量最高未知位，把小根界降到可行范围；截断 LCG 则把每个状态的低 256 位视为小误差，利用多次输出建立近似线性关系并交给 LLL。两阶段都要在模 $p$ 的精确递推与整数小量之间保持清晰区分。
