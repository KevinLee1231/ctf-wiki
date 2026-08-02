# TSGCTF2024 Easy?? ECDLP

## 题目简述

服务允许选一个至少 200 位的素数 $p$，随后给出 250 组定义在 $K=\mathbb Q_p$、精度 8 上的椭圆曲线与点：

$$Q=[s]P,\qquad 0\le s<2^{200\cdot8}$$

每组都必须返回正确标量 $s$ 才能得到 flag。曲线的 $j$ 不变量被随机调整到 valuation $-2$ 或 $-3$，因此曲线具有坏约化；直接把它们当作普通光滑 $E(\mathbb F_p)$ ECDLP 会失败。利用思路是把坏约化拆成“奇异约化群上的离散对数”和“形式群中的 $p$ 进离散对数”。

## 解题过程

### 1. 选择适合三类奇异约化的素数

奇异三次曲线的非奇异部分可能同构于阶为 $p$、$p-1$ 或 $p+1$ 的群。solver 预先生成素数：

```text
p = 1558924682799949478580620526034531580383503455984448793743467
```

生成时让 $p-1$ 与 $p+1$ 都光滑：$p-1$ 由小素因子乘积构造，同时检查 $p+1$ 的最大素因子也足够小。这样分裂或非分裂节点约化中的离散对数都可由 Pohlig–Hellman 快速完成；尖点约化的群阶为 $p$，其加法结构更直接。

### 2. 缩放到最小整模型并约化

反序列化曲线和点后，根据不变量 valuation 计算共同缩放量：

```sage
t = min(
    E.c4().valuation() // 4,
    E.c6().valuation() // 6,
    E.discriminant().valuation() // 12,
)
u = p ** (-t)
E = E.scale_curve(u)
P = E.point((P[0] * u**2, P[1] * u**3, 1))
Q = E.point((Q[0] * u**2, Q[1] * u**3, 1))
```

缩放后所有 Weierstrass 系数 valuation 非负，可以约化到 $\mathbb F_p$。由于 $j$ 的负 valuation，约化曲线是奇异三次曲线。

求解偏导为零得到奇点 $(x_s,y_s)$，再分解奇点处的切锥：

$$
f(x,y)+(x-x_s)^3
=\bigl(y-y_s-\alpha(x-x_s)\bigr)
 \bigl(y-y_s-\beta(x-x_s)\bigr)
$$

按 $\alpha,\beta$ 分类：

| 情况 | 约化类型 | 非奇异部分群阶 |
| --- | --- | --- |
| $\alpha=\beta$ | 尖点 cusp | $p$ |
| $\alpha\ne\beta$ 且两者在 $\mathbb F_p$ | 分裂节点 | $p-1$ |
| 切线只在二次扩域分裂 | 非分裂节点 | $p+1$ |

节点情形可用两条切线之比构造到乘法群的参数：

$$
\psi(x,y)=
\frac{y-y_s-\alpha(x-x_s)}
     {y-y_s-\beta(x-x_s)}
$$

尖点则用相应有理参数映射到加法群。

### 3. 用形式群恢复 $p$ 进部分

设奇异约化的非奇异群阶为 $n\in\{p,p-1,p+1\}$。乘以 $n$ 后，$nP$ 与 $nQ$ 约化为单位元，落入形式群。令 $\ell$ 为形式群对数，则：

$$
\frac{\ell(nQ)}{\ell(nP)}\equiv s\pmod{p^{\mathrm{PREC}}}
$$

Sage 实现为：

```sage
ell = E.formal_group().log().polynomial()
dlog_on_k = ZZ(
    ell((n*Q)[0] / (n*Q)[1]) /
    ell((n*P)[0] / (n*P)[1])
)
```

若 $P$ 自身约化到无穷远，说明它已经在形式群中，可以直接取 $\ell(Q)/\ell(P)$。

尖点且 $P$ 不在形式群时，$n=p$ 会让除法损失一位 $p$ 进精度；与第一题相同，对残差点再修正：

```sage
ans = ZZ(log(p * Q) / log(p * P))
R = Q - ans * P
ans += ZZ(log(R) / log(p * P)) * ZZ(p)
```

### 4. 节点情形用 CRT 合并

若约化为节点，先在约化群上求：

```sage
x = psi(P_reduced)
y = psi(Q_reduced)
dlog_on_F = y.log(x)  # s mod (p-1) 或 s mod (p+1)
```

再把该结果与形式群给出的 $s\bmod p^8$ 合并：

```sage
ans = ZZ(crt(dlog_on_F, dlog_on_k, n, p**8))
```

$\gcd(n,p)=1$，且 CRT 模数覆盖服务端生成 secret 的范围，因此恢复值唯一。每轮提交前必须验证：

```sage
assert ans * original_P == original_Q
```

对 250 组曲线分别按“形式群点、尖点、分裂节点、非分裂节点”分流并提交，最终得到：

```text
TSGCTF{BAD r3duCT1oN IS n0t 5o bAD!}
```

## 方法总结

本题刻意让曲线具有坏约化，因此奇异约化不是需要绕开的异常，而是泄露标量的主通道。最小化模型后，尖点、分裂节点和非分裂节点分别对应阶 $p,p-1,p+1$ 的易处理群；选择让 $p\pm1$ 光滑的素数可统一解决剩余域离散对数。形式群对数恢复 $p$ 进部分，节点情形再用 CRT 与约化群结果合并。完整 solver 的关键是覆盖所有约化分支并在每轮用 $[s]P=Q$ 做强验证。
