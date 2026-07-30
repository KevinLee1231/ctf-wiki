# L3akCTF 2025 Dual Mersenne Twister DRBG Writeup

## 题目简述

题目同时初始化两个 Python `random.Random` 实例。每次取出两个 32 位 MT19937 输出 $v_1,v_2$，返回

$$
L_i=\left(v_1+\left(v_2\oplus\operatorname{ror}_1(v_2)\right)\right)\bmod 2^{32}.
$$

选手可以指定一次泄露数量，随后必须分别提交两个生成器接下来的 624 个输出。两个种子都来自 `os.urandom(32)`，因此爆破种子不可行；真正的弱点是 MT19937 的状态转移和输出 temper 都是 $\mathrm{GF}(2)$ 上的线性变换，而组合输出仍泄露了足够多的低位线性关系。

## 解题过程

### 从模加法中取出两条线性约束

记

$$
a=v_1,\qquad c=v_2\oplus\operatorname{ror}_1(v_2),\qquad y=a+c\bmod 2^{32}.
$$

按最低位在前编号，有

$$
c_0=b_0\oplus b_1,\qquad c_1=b_1\oplus b_2.
$$

最低位没有进位，所以

$$
y_0=a_0\oplus c_0.
$$

第二位满足

$$
y_1=a_1\oplus c_1\oplus(a_0\land c_0).
$$

表面上第二式出现了非线性的进位项，但已知 $y_0=a_0\oplus c_0$ 后可以分情况消掉：

- 若 $y_0=1$，则 $a_0\ne c_0$，进位为 0；
- 若 $y_0=0$，则 $a_0=c_0$，进位等于 $a_0$。

因此每个组合输出都能提供两条关于两个 MT19937 状态的线性方程：

$$
\begin{aligned}
y_0&=a_0\oplus b_0\oplus b_1,\\
y_1&=
\begin{cases}
a_1\oplus b_1\oplus b_2,&y_0=1,\\
a_1\oplus b_1\oplus b_2\oplus a_0,&y_0=0.
\end{cases}
\end{aligned}
$$

官方脚本请求 20000 个输出，并只提交每个输出的最低两位：

```python
io.sendline(b"20000")
L = ast.literal_eval(io.recvline().rstrip().decode())

for v in L:
    solver.submit("?" * 30 + str((v >> 1) & 1) + str(v & 1))
```

两个 MT19937 一共有

$$
2\times624\times32=39936
$$

个状态位，而 20000 个样本恰好给出 40000 条方程，数量上足以唯一确定联合状态。

### 建立 MT19937 的符号状态

将两个生成器最初的 624 个 32 位状态字展开成 39936 个布尔未知量。对每次 twist 和 temper，不保存具体整数，而是保存“当前每一位由哪些初始状态位异或得到”的布尔向量。

MT19937 的 twist 为

$$
x=(s_i\mathbin{\&}\texttt{UMASK})\mid
(s_{i+1}\mathbin{\&}\texttt{LMASK}),
$$

$$
s_i'=s_{i+m}\oplus(x\gg1)\oplus
\begin{cases}
A,&x_0=1,\\
0,&x_0=0.
\end{cases}
$$

其中最后一项虽然写成条件选择，但掩码 $A$ 是常量，因此每个目标位仍只是 $x_0$ 的线性复制。随后四步 temper 也只有移位、掩码和异或，所以整个输出位都能表示为初始状态位的线性组合。

对每个泄露值，官方实现把两个生成器对应的符号输出拼在同一行中，再按已知的 $y_0$ 选择第二位的线性化形式。最终得到

$$
M\mathbf{s}=\mathbf{b}\pmod 2.
$$

矩阵规模接近 $40000\times39936$，普通 Python 高斯消元会非常慢。官方解法通过 C 扩展调用 M4RI 的 `mzd_solve_left` 在 $\mathrm{GF}(2)$ 上求解，再用恢复出的最终状态预测两个生成器后续各 624 个 32 位输出。

### 提交预测

求解器需要保持与服务端相同的 twist 位置。恢复状态后，先把符号状态推进到 20000 次泄露结束的位置，再依次生成：

```python
next_r1 = [mt1.getrandbits(32) for _ in range(624)]
next_r2 = [mt2.getrandbits(32) for _ in range(624)]

io.sendlineafter(b"r1:", ",".join(map(str, next_r1)).encode())
io.sendlineafter(b"r2:", ",".join(map(str, next_r2)).encode())
```

两组预测全部正确后得到：

```text
L3AK{Huh_m3rs3nne_tw1s7er5_4re_5urpri5in6ly_l1ne4r!}
```

## 方法总结

这题不是种子强度问题，而是线性 PRNG 的状态泄露问题。关键是不要试图拆出每个完整输出：模加法的最低位无进位，第二位的进位又能由已知最低位分情况线性化，于是每个样本稳定提供两条 $\mathrm{GF}(2)$ 方程。样本数略高于两个 MT19937 的联合状态位数后，就可以用高性能布尔矩阵库恢复全部状态并预测后续输出。
