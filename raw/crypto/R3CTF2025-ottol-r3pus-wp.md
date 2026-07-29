# ottol r3pus

## 题目简述

服务在每轮生成两个 512 位素数 $p,q$，并输出：

$$
\begin{aligned}
x_i&=u_i+r_{1,i}p,\\
y_i&=v_i+r_{2,i}q,\\
u_i+v_i&=s_i2^{128}+e_i,
\end{aligned}
$$

其中 $s_i<2^{64}$、$e_i<2^{128}$，所以 $u_i+v_i<2^{192}$；而 $x_i,y_i$ 通常接近 1024 位。正常路线要求在 16 次菜单操作中猜中 10 个 `secret`，但菜单还有一个后门：若输入恰好等于本轮固定的素数 $p$，服务会直接输出 flag。

因此只需用前 15 轮收集 $(x_i,y_i)$，恢复 $p$，再把它作为第 16 次菜单选项提交。

## 解题过程

### 把样本写成向量关系

令：

$$
\boldsymbol{x}=\boldsymbol{u}+p\boldsymbol{r}_1,\qquad
\boldsymbol{y}=\boldsymbol{v}+q\boldsymbol{r}_2.
$$

虽然每个 $u_i$ 本身可达 512 位，但 $\boldsymbol{u}+\boldsymbol{v}$ 只有约 192 位。这个异常短的关系使得隐藏的随机系数向量 $\boldsymbol r_1,\boldsymbol r_2$ 可以通过正交格辨认出来。

构造 15 行的格基：

```python
M = identity_matrix(15).stack(x_v).stack(y_v).transpose()
```

其任意整数行组合 $\boldsymbol w$ 对应：

$$
(\boldsymbol w,\ \langle\boldsymbol w,\boldsymbol x\rangle,\
\langle\boldsymbol w,\boldsymbol y\rangle).
$$

对 `M` 做 LLL 后，前 13 个向量的范数会明显小于最后两个。它们的末两维趋于为零，并落在 $\boldsymbol r_1$、$\boldsymbol r_2$ 的共同正交子空间中；最后两个向量则因必须补足线性无关方向而陡然变长。

### 从正交补恢复随机系数

去掉每个短向量最后两维，将前 13 行组成矩阵 `r_o`：

```python
vs = [row[:-2] for row in M.LLL()]
r_o = Matrix(vs[:-2])
r_l = r_o.right_kernel_matrix().LLL()
```

`r_o` 的整数右核是二维格，包含真实的 $\boldsymbol r_1$ 与 $\boldsymbol r_2$。LLL 给出的基不一定恰好按原顺序返回两者；实践中第一行通常是二者之差一类的短组合，第二行通常接近其中一个真实向量。因此依次测试：

```python
candidates = [
    r_l[1] - r_l[0],
    r_l[1],
    r_l[1] + r_l[0],
]
```

对于正确的 $\boldsymbol r_1$，有：

$$
\frac{x_i}{r_{1,i}}=p+\frac{u_i}{r_{1,i}}\approx p.
$$

所以所有 `round(x_i / r1_i)` 都集中在同一个整数附近；错误候选得到的比值不会集中。由于舍入误差很小，只需在该中心附近枚举十几个整数，检查 512 位素数即可得到 $p$。

### 利用菜单后门

前 15 轮各选择一次游戏、保存 $x_i,y_i$，随意提交错误答案。恢复 $p$ 后，第 16 次不要选择 `1` 或 `2`，直接发送十进制的 $p$。程序会进入：

```python
elif coi == p:
    print("Wtf? You known the real secret number!")
    print(flag)
```

这样不需要实际恢复任意一轮的 64 位 `secret`。

## 方法总结

本题把两个近似公因子关系耦合在一起，并用 $u_i+v_i$ 的短和制造了可由 LLL 识别的低维结构。解题时最重要的信号是约化基在第 14、15 行出现明显范数断层：前 13 行给出隐藏随机系数的共同正交补，再求整数核即可回到由 $\boldsymbol r_1,\boldsymbol r_2$ 张成的二维格。

求解脚本及该范数断层的解释可参考 [Connor McCartney 的 R3CTF 2025 记录](https://connor-mccartney.github.io/cryptography/other/R3CTF2025)；正文已经给出所需矩阵、候选向量和素数判定流程。
