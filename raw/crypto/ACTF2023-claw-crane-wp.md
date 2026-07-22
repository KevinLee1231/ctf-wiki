# claw crane

## 题目简述

服务连续进行 256 轮夹娃娃。每轮先给出一个 $16\times16$ 网格坐标，选手提交不超过 100 个 `W/S/A/D`；程序从 $(0,0)$ 模拟移动，只有最终坐标正确才继续。前 64 个移动会按 `W/S/A/D -> 0/1/2/3` 解释为 128 位四进制整数 $v$，并生成

$$
c=\operatorname{MD5}\bigl((\mathrm{seed}+v)\bmod 2^{128}\bigr),
\qquad
\mathrm{seed}'=(\mathrm{seed}+c+1)\bmod 2^{128}.
$$

随后需要提交 $0<p,q<2^{64}$。程序计算

$$
\Delta=\left|cq-p\cdot2^{128}\right|,
$$

从 $\Delta$ 的二进制表示中随机抽取一位，抽到 `0` 才得 10 分。若位串短于 $128+\mathrm{bless}$ 位，程序会在末尾补零；失败令 `bless += 16`，成功则将其清零。最终至少需要 2220 分，即 256 轮中成功 222 次。

## 解题过程

### 用连分数缩小误差

要让 $\Delta$ 尽量小，本质上需要在 $p,q<2^{64}$ 的约束下寻找

$$
\frac{p}{q}\approx\frac{c}{2^{128}}.
$$

连分数的渐近分数会在给定分母规模下提供极好的有理逼近，因此可以对 $c/2^{128}$ 做连分数展开，在分母即将超过 $2^{64}$ 时取最后一个合法渐近分数。这个结论及渐近分数的递推、误差界可参考 [OI Wiki 的连分数说明](https://oi-wiki.org/math/number-theory/continued-fraction/)；本题实际使用的是“在分母上界内选逼近误差最小的分数”这一点，而不是依赖外链中的现成脚本。

官方脚本的核心实现如下：

```python
def fraction_range(num, den, upper):
    coeffs = []
    while den:
        coeffs.append(num // den)
        num, den = den, num % den
        p, q = transfer_frac(coeffs)
        if q > upper:
            break
    return transfer_frac(coeffs[:-1])
```

普通渐近分数得到的 $\Delta$ 约为 64 位。随机低位本身约有一半为零，再补到 128 位，单轮成功率仍不够。官方解进一步令 $p\leftarrow p+2^{63}$，使误差中出现一个约 $2^{191}$ 的高位项；在符号合适时，高位与原低位误差之间形成大段零位。脚本会检查实际二进制零位比例，若不足 $220/256$ 就断开重连，直到找到零位足够多的一组 $c,p,q$。这里不能只按期望值估计，必须以实际 `bin(delta)` 的零位数量筛选。

### 重放同一个 MD5 输出

找到高质量的 $c$ 后，需要让后续轮次继续得到同一个输出。设当前输入为 $v_t$、输出为 $c_t$，则

$$
\mathrm{seed}_{t+1}=\mathrm{seed}_t+c_t+1\pmod{2^{128}}.
$$

若选择

$$
v_{t+1}=v_t-c_t-1\pmod{2^{128}},
$$

便有

$$
\mathrm{seed}_{t+1}+v_{t+1}
=\mathrm{seed}_t+v_t\pmod{2^{128}},
$$

所以 MD5 输入和输出都保持不变。将 $v_{t+1}$ 写成 64 位四进制数即可还原为恰好 64 个移动。网格每个方向都会在边界处截断，因此执行这 64 步后，再追加横纵方向最多 30 步就能到达任意目标坐标，总长度不超过 94，满足 100 步限制。

核心重放关系可直接写成：

```python
def replay_value(chaos, previous_value):
    return (previous_value - chaos - 1) % (1 << 128)
```

后续各轮复用同一组 $c,p,q$。即使偶尔失败，`bless` 还会再补 16 个零，提高下一轮成功率；累计至少 222 次成功后即可得到 flag：

```text
ACTF{C0Nt1nu3d_Fract1on&Mor3_zer0s&Replay_it}
```

## 方法总结

- 核心技巧：用连分数在 $p,q<2^{64}$ 下逼近 $c/2^{128}$，再通过人为加入高位项制造长零区，并筛选实际零位比例。
- 识别信号：随机过程若使用 `hash(state + controllable)` 且状态按已知输出线性更新，应立即检查能否让下一轮哈希输入保持不变。
- 复用要点：交互题不能只优化单轮概率；还要同时核对输入编码、移动长度、状态更新和失败后的补偿机制，确认高质量样本可以跨轮复用。
