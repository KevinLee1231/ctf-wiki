# DownUnderCTF 2022 1337crypt v3 Writeup

## 题目简述

题目选择约 780 位整数 $x$，公开 1337 位精度的三组三角函数关系：

$$
\sin(\alpha_1x)=\beta_1,
\quad\cos(\alpha_2x)=\beta_2,
\quad\tan(\alpha_3x)=\beta_3.
$$

flag 被按位左移并补随机低位，直到与 $x$ 位长相同，再生成 $c=x\oplus m$。由于 flag 格式以 `DUCTF{` 开头，$m$ 的最高 47 位已知，从而也泄露 $x$ 的最高 47 位。

## 解题过程

三角函数不能在整个实数域直接求逆，但具有周期性。枚举正弦、余弦的少量相位分支后，可以写成：

$$
\alpha_1x\equiv r_1\pmod{2\pi},\qquad
\alpha_2x\equiv r_2\pmod{2\pi},\qquad
\alpha_3x\equiv r_3\pmod{\pi}.
$$

其中 $r_1$ 来自 $\arcsin(\beta_1)$ 或其对称分支，$r_2=\pm\arccos(\beta_2)$，$r_3=\arctan(\beta_3)$。将所有高精度实数乘 $S=2^{1337}$ 并取整，有限精度会产生相对模数很小的误差 $k_i$：

$$
S(\alpha_i)x-S(r_i)\equiv k_i\pmod{S(T_i)},
$$

其中 $T_1=T_2=2\pi$、$T_3=\pi$。

由已知 flag 前缀计算 $x$ 的已知高位 $\bar x$，写成 $x=\bar x+x'$。官方解法使用误差界 $B=2^{800}$，构造五维格：

$$
\begin{pmatrix}
S(2\pi)&0&0&0&0\\
0&S(2\pi)&0&0&0\\
0&0&S(\pi)&0&0\\
S(\alpha_1)&S(\alpha_2)&S(\alpha_3)&B/2^{733}&0\\
S(\alpha_1)\bar x-S(r_1)&S(\alpha_2)\bar x-S(r_2)&S(\alpha_3)\bar x-S(r_3)&0&B
\end{pmatrix}.
$$

正确的整数线性组合产生形如 $(k_1,k_2,k_3,Bx'/2^{733},B)$ 的短向量。LLL 后寻找末坐标等于 $B$ 的行，便可从倒数第二坐标读出 $x'$。

```python
reduced = M.LLL()
for row in reduced:
    if row[-1] == B:
        x = xbar + row[-2] // (B / 2^733)
        padded = int(c) ^ int(x)
        break
```

计算 $m=c\oplus x$ 后不断右移，直到字节串同时以 `DUCTF{` 开头并以 `}` 结尾，得到：

```text
DUCTF{1337_tr1g0n0m3try_4nd_l4tt1c3_sk1llz_1f8a84e9a9cdef57}
```

## 方法总结

高精度周期函数值可以转化为带小舍入误差的模线性关系，再交给 HNP/格约简处理。关键步骤是枚举正确相位、选择足够大的整数缩放、利用已知明文前缀缩小秘密范围，并对格坐标按各未知量的界进行平衡。恢复出的 $x$ 还必须通过 flag 格式和异或关系验证，不能只凭某个短向量形态认定成功。
