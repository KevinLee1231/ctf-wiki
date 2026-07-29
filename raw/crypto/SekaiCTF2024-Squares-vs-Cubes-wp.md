# Squares vs. Cubes

## 题目简述

题目生成三个 $512$ 位素数 $p,q,r$，并令：

$$
N=e=pqr,
$$

$$
\varphi(N)=(p-1)(q-1)(r-1),\qquad d=e^{-1}\bmod\varphi(N).
$$

把带随机后缀的 flag 整数记作 $f$，再定义 $\zeta=q+rf$。服务通过一轮 RSA oblivious transfer 返回：

$$
m_0=(v-x_0)^d+p^2+\zeta^2\pmod N,
$$

$$
m_1=(v-x_1)^d+p^3+\zeta^3\pmod N.
$$

看似只能在“平方”和“立方”中选择一个，但两个响应共享相同的 $p,q,r,f$，可构造公共多项式根。

## 解题过程

### 选择退化的 OT 输入

直接提交 $v=x_0$，第一项中的 RSA 部分变为 $0$：

$$
m_0=p^2+\zeta^2\pmod N.
$$

第二项为：

$$
m_1=(x_0-x_1)^d+p^3+\zeta^3\pmod N.
$$

由于 $ed\equiv1\pmod{\varphi(N)}$，对后一式的 RSA 部分再取 $e=N$ 次幂，可恢复 $x_0-x_1$。

### 用模 $p$ 公共根分解 $N$

模 $p$ 时，$p^2$ 和 $p^3$ 消失，因此 $\zeta$ 同时满足：

$$
\zeta^2-m_0\equiv0\pmod p,
$$

$$
x_0-x_1+(\zeta^3-m_1)^e\equiv0\pmod p.
$$

不能直接展开次数为 $N$ 的多项式。改在商环

$$
\mathbb{Z}_N[\zeta]/(\zeta^2-m_0)
$$

中做快速幂，第二个多项式会约化为一次式 $a\zeta+b$。归一化为 $\zeta+c$ 后，公共根意味着 $\zeta\equiv-c\pmod p$，所以：

$$
p=\gcd(N,c^2-m_0).
$$

对应 Sage 代码为：

```python
def polypow(base, exp, modulus):
    return (modulus.parent().quo(modulus)(base) ** exp).lift()

z = polygen(Zmod(N))
linear = polypow(z**3 - m1, N, z**2 - m0) + x0 - x1
c = ZZ(linear.monic()[0])
p = gcd(N, c**2 - m0)
```

### 恢复 $\zeta$ 与剩余素因子

已知 $p$ 后，在完整模 $N$ 方程中保留 $p^2,p^3$，再次把高次式约化为一次式，即可得到 $\zeta=q+rf$：

```python
linear = polypow(
    z**3 + p**3 - m1,
    N,
    z**2 + p**2 - m0,
) + x0 - x1
zeta = ZZ(-linear.monic()[0])
```

令 $M=qr=N/p$。由：

$$
q\zeta=q(q+rf)\equiv q^2\pmod{qr}
$$

可得：

$$
q(q-\zeta)\equiv0\pmod M.
$$

flag 以 `SEKAI{` 开头，因此已知 $f$ 的高位。利用 $\zeta\approx rf=Mf/q$ 得到 $q$ 的高位近似 $q_0\approx f_0M/\zeta$。把 $q=q_0+y$ 代入上式，对多项式

$$
(q_0+y)(q_0+y-\zeta)\pmod M
$$

做 Coppersmith 小根求解，即可恢复 $q$；再计算 $r=M/q$ 和：

$$
f=\frac{\zeta-q}{r}.
$$

将 $f$ 转成字节后去掉随机后缀，保留至第一个 `}` 即为 flag。若本轮随机参数导致 $q+rf\ge N$ 或近似误差不满足界，官方题解建议重新连接获取一组参数。

官方 notebook 输出中去除随机后缀后的 flag 为：

```text
SEKAI{this_challenge_was_originally_called_oblivion_but_there_was_an_ACSC_crypto_also_called_that}
```

## 方法总结

- 核心技巧：主动选择 $v=x_0$，把 OT 输出转为具有公共根的平方/立方多项式；先借公共根分解 $N$，再用已知明文高位做 Coppersmith。
- 识别信号：多个响应共享复合模数的秘密素因子和同一隐藏变量，且不同次数项在模某个素因子时会消失。
- 复用要点：面对极高次多项式不应直接展开，应先在低次关系生成的商环中约化；得到素因子后，还要重新使用完整模 $N$ 方程恢复隐藏量，不能停在模 $p$ 的剩余类。
