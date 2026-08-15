# BSidesAlgiers2025 - Very Cool Cryptosystem

## 题目简述

题目是一个自定义矩阵加密系统，`dist/chall.sage` 将 flag 分两段：

- `flag[:30]` 参与公钥生成；
- `flag[30:]` 用随机共轭矩阵进行一次矩阵加密后发布。

已知公开量：
- `out.txt`（`n,b_s,Dr` 的部分元，`A,eps,U_`）
- `dist/chall.sage`（公钥/私钥构造与加密公式）
- `solution/sol_chall.sage`（可执行复原链）

## 解题过程

### 1. 按源码提炼加密方程

`dist/chall.sage` 定义：

```python
p = random_prime(2**100)
q = random_prime(2**100)
n = p * q
SIZE = 4
pub, b_s, D = gen_pub(flag[:30])
U_, eps = encrypt(pub, flag[30:])
```

其中
- `D` 是私钥矩阵相关对象（`priv_key` 构造）；
- `Dr = D^5`；
- 输出时仅记录 `Dr[0][1],Dr[1][2],Dr[2][3]`、`A`、`eps`、`U_`。

`encrypt` 中关键变换是共轭：

$$
\mathrm{eps}=\gamma^{-1}A\gamma,\quad
U_ = kUk,\quad k=\gamma^{-1}B\gamma,\ \gamma = D^s,\ s\in[1,2^{10}]
$$

这里 `A,B,Dr` 来自公钥 `pub`，`eps,U_` 为题目给定输出。

### 2. 用 `b_s` 和 `out.txt` 建立代数约束并恢复 `a` 与后缀明文

`solution/sol_chall.sage` 读取 `out.txt` 后，先用三条导出方程约束变量：

```python
M_0_1, M_1_2, M_2_3
```

设

$$
\text{known}= \text{int}(b"shellmat")\ll 176,\quad \text{dbl}=(\text{known}+x)^6
$$

并构造三条多项式方程（变量 `x`,`y`，`k=5,e_1=5`）：

$$
\begin{aligned}
f_1(y) &= (dbl^5 y + b_0)\cdot \sum (y+dbl)^j((y^2)+dbl)^{k-1-j}-M_{01}\\
f_2(y) &= (y(dbl+b_1)^5+b_2)\cdot \sum (y^2+dbl)^j((y^3)+dbl)^{k-1-j}-M_{12}\\
f_3(y) &= \big(y(y(dbl+b_3)^5+b_4)+b_5\big)\cdot \sum (y^3+dbl)^j((y^4)+dbl)^{k-1-j}-M_{23}
\end{aligned}
\mathrm{mod}\ n
$$

这些方程源自已知 $Dr=U^{-1}\epsilon U$ 的约束（见脚本注释与实现路径）。用 Gröbner 基求解 $I=\langle f_1,f_2,f_3\rangle$，得到变量 `y`（对应 `a`）。得到 $a$ 后，再代回 $f_0$ 解第二变量 $x$ 并转字节，得到后续 flag 段 `m`。

### 3. 重建私钥并反共轭恢复明文矩阵

从 `flag1 = b'shellmat'+m` 重入 `priv_key` 复现私钥矩阵 `D`：

```python
D, b_s = priv_key(flag1)
assert (D^5)[0][1] == M_0_1
assert (D^5)[1][2] == M_1_2
assert (D^5)[2][3] == M_2_3
```

成立则可计算：

$$
\lambda = D^{-1}\cdot eps\cdot D,\quad U=\lambda U_\text{enc}\lambda
$$

最后把矩阵 `U` 展开为字节并拼接到 `flag = b"shellmat"+m` 后面。

### 4. 可执行链条

```bash
sage solution/sol_chall.sage
```

脚本自带输出：
- `m` 可能明文候选（`isinstance` 过滤可 ASCII 字节串）；
- 复原 `U` 后拼出完整 flag。

公开复盘给出的完整结果为：

`shellmates{CayleyPurser_without_pub_key?_grobner?_anyway_hope_u_used_the_right_monomial_ordering_isthisenough}`

该结果与仓库 solver 的“恢复 `a` 和前 30 字节、重建 `D`、反共轭解出剩余矩阵块”结构一致。另一种公开实现会先在模 $p$、模 $q$ 的域上消元并用 CRT 合并，再枚举很小的共轭指数；[Very Cool Cryptosystem 复盘](https://medium.com/@yanisderiche22/very-cool-cryptosystem-6fcc15581ca0) 中的这个替代路径说明了直接在合数模环上求 Gröbner 基失败时应如何降级。正文保留官方 solver 的主线，外链只作替代算法与最终结果的来源核对。本轮未重新执行可能耗时的 Sage 求解。

## 方法总结

- 核心技巧：将 `out.txt` 的稀疏公开约束与加密中的幂共轭结构联立，先解出控制参数，再反向生成可验一致性的私钥共轭逆元。
- 识别信号：看到 `X^(-1)AX`、`XKX` 类输出时，优先考虑构造同位多项式 + Groebner/代数验证，优先用 `Dr` 的局部条目约束未知变量。
- 复用要点：该题关键闭环是“已知 `out.txt` 的三条 Dr 条件 + `b_s` + `priv_key` 结构一致性检查”，缺失其一将导致误解。
