# DownUnderCTF 2023 Number Theoretic Flag Checker Writeup

## 题目简述

校验器先把 256 字节输入的每个系数乘以 61，再在 $\mathrm{GF}(7937)$ 上执行一套 NTT 风格的变换。变换结果随后被当作一个次数不超过 255 的多项式系数，并在 $x=1,2,\ldots,255$ 处求值，与程序内置的 255 个点比较。

直接逆向每一级蝶形操作很繁琐。更清晰的做法是先从 255 个求值点恢复变换后的多项式，再利用变换对应的二次因子和中国剩余定理还原输入多项式。

## 解题过程

令变换后的 256 个值为 $a_0,\ldots,a_{255}$，校验器实际构造

$$
f(x)=\sum_{i=0}^{255}a_ix^i
$$

并给出 $f(1),\ldots,f(255)$。由于只缺少 $f(0)$，可以在有限域 $F=\mathrm{GF}(7937)$ 中枚举 $x_0=f(0)$，然后对 256 个点

$$
(0,x_0),(1,f(1)),\ldots,(255,f(255))
$$

做拉格朗日插值。每个候选都会唯一确定 $f$，其系数正是需要逆变换的 256 个域元素。

源码使用 $N=256$、$\zeta=2805$。把系数两两组成次数小于 2 的多项式后，它们可以视为原多项式在下列 128 个二次模数下的余式：

$$
M_i(X)=X^2-\zeta^{2\operatorname{br}(i)+1},\qquad 0\le i<128,
$$

其中 $\operatorname{br}(i)$ 是 7 位 bit-reversal。所有 $M_i$ 的乘积为 $X^{256}+1$，所以对这 128 个余式执行多项式 CRT，就能在商环

$$
F[X]/(X^{256}+1)
$$

中重建变换前的多项式。

官方 Sage 脚本的核心如下：

```sage
q = 7937
F = GF(q)
P.<X> = PolynomialRing(F)

def bit_reverse_7(x):
    result = 0
    for i in range(7):
        result = (result << 1) | ((x >> i) & 1)
    return result

zeta = F(2805)
moduli = [X^2 - zeta^(2 * bit_reverse_7(i) + 1) for i in range(128)]

for x0 in range(q):
    samples = [(0, F(x0))]
    samples += [(i + 1, F(y)) for i, y in enumerate(points)]
    transformed = P.lagrange_polynomial(samples)

    coeffs = list(transformed)
    residues = [P(coeffs[i:i + 2]) for i in range(0, 256, 2)]
    original = crt(residues, moduli)

    candidate = [int(c) // 61 for c in list(original)]
    if all(0 < value < 256 for value in candidate):
        print(bytes(candidate).decode())
        break
```

flag 使用可打印 ASCII，故最大字符系数满足 $126\times61=7686<7937$；正确候选不会在乘 61 时绕回有限域，可以直接做整数除法。最终得到：

```text
DUCTF{fft_alg0r1thm_1d3ntif1ed_372ca4e0a11dabbc}
```

## 方法总结

本题把“多项式插值”和“NTT 的 CRT 表示”叠在了一起。先枚举唯一缺失的 $f(0)$，可将不完整求值问题补成标准插值；再识别蝶形变换对应的二次模数，用多项式 CRT 一次性完成逆变换。与逐条逆执行汇编相比，这种从代数结构出发的还原更短，也更容易验证。
