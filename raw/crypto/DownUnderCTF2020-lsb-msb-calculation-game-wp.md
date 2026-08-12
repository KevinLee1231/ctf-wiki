# DownUnderCTF 2020 - LSB||MSB Calculation Game

## 题目简述

服务使用线性同余生成器：

$$
x_{i+1}=ax_i+c\pmod M,
$$

但每轮只输出状态的最低 20 位和最高 20 位。玩家每轮可提交最多 5 个猜测，23 轮中命中超过 10 次即可获得 flag。$M$ 与 $a$ 公开，增量 $c$ 和完整状态未知；决定性问题是用格恢复被截断的中间位并预测后续输出。

## 解题过程

令 $l=\operatorname{bitlen}(M)$、$t=20$，把状态写成：

$$
x_i=2^{l-t}w_i+2^ty_i+z_i,
$$

其中 $w_i$、$z_i$ 分别是已知的最高和最低 $t$ 位，$y_i$ 是未知中段。为了消去未知 $c$，使用相邻差分：

$$
x'_i=x_{i+1}-x_i,qquad x'_i\equiv ax'_{i-1}\pmod M.
$$

同样定义 $w'_i=w_{i+1}-w_i$、$y'_i=y_{i+1}-y_i$、$z'_i=z_{i+1}-z_i$。对于 $k$ 个差分，构造：

$$
L=
\begin{bmatrix}
M&0&\cdots&0\\
a&-1&\cdots&0\\
a^2&0&\ddots&0\\
\vdots&\vdots&&-1
\end{bmatrix}.
$$

LLL 约化得到短基 $B$ 后，由 $B\mathbf{x}'\equiv0\pmod M$ 和状态分解式可近似取整求出 $B\mathbf{y}'$，再解线性方程恢复所有完整差分：

```sage
def recover_differences(W, Z, a, M, trunc=20):
    count = len(Z) - 1
    Wp = [W[i + 1] - W[i] for i in range(count)]
    Zp = [Z[i + 1] - Z[i] for i in range(count)]

    lattice = [[a^i] + [0] * (i - 1) + [-1] + [0] * (count - 1 - i)
               for i in range(1, count)]
    lattice.insert(0, [M] + [0] * (count - 1))
    B = Matrix(lattice).LLL()

    known = 2^(M.nbits() - trunc) * B * vector(Wp) + B * vector(Zp)
    rounded = vector([
        (round(RR(value) / M) * M - value) / 2^trunc
        for value in known
    ])
    Yp = list(B.solve_right(rounded))

    return [ZZ(2^(M.nbits() - trunc) * Wp[i]
               + 2^trunc * Yp[i] + Zp[i])
            for i in range(count)]
```

若最后已恢复 $x'_{k-1}$，则下一差分满足 $x'_k\equiv ax'_{k-1}\pmod M$。从它的高低位变化可以计算下一输出。唯一歧义是实际整数差可能为该模值或该模值减 $M$；再考虑官方实现中可能出现的 1 位取整误差，提交四个候选即可覆盖：

```sage
def next_guesses(xprime, w, z, a, M, trunc=20):
    def encode(delta):
        next_z = (delta + z) % 2^trunc
        next_w = ((delta >> (M.nbits() - trunc)) + w) % 2^trunc
        return (next_z << trunc) + next_w

    delta = (a * xprime) % M
    positive = encode(delta)
    negative = encode(delta - M)
    return [positive, negative, positive + 1, negative + 1]
```

前两轮可故意猜错以取得真实输出；之后每轮把新增的 $w_i,z_i$ 加入格中重新恢复差分，并提交上述候选。命中次数超过 10 后得到：

```text
DUCTF{y0u_4r3_A_m4st3r_1n_gu3ss1ng_y0u_w1ll_g0_v3ry_f4r!_:)}
```

## 方法总结

截断 LCG 的核心是把已知高低位和未知中段写成同一整数分解，再用差分消除未知增量。LLL 恢复的是相邻状态差而非完整状态，但这已足够预测下一次泄漏；若从模同余还原到有符号整数存在歧义，应利用服务允许的多次猜测覆盖候选，而不是强行假定差分为正。
