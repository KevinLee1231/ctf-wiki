# SUCTF2026-Restaurant

## 题目简述

题目实现了一套基于热带半环的矩阵签名。这里的运算不是普通线性代数，而是 min-plus 运算：

$$
a\oplus b=\min(a,b),\qquad a\otimes b=a+b,
$$

$$
(A\otimes B)_{i,j}=\min_k(A_{i,k}+B_{k,j}).
$$

服务启动时随机生成私钥矩阵 $X\in[0,255]^{8\times7}$、$Y\in[0,255]^{7\times8}$，并保存 $T=X\otimes Y$。消息经 SHA3-512 映射为 $8\times8$ 矩阵 $M$。一次正常签名随机选择 $U,V$，输出：

$$
\begin{aligned}
A&=(M\otimes X)\oplus U,& B&=(Y\otimes M)\oplus V,\\
P&=X\otimes V,& R&=U\otimes Y,& S&=U\otimes V.
\end{aligned}
$$

验签计算

$$
Z=(M\otimes T\otimes M)\oplus(M\otimes P)\oplus(R\otimes M)\oplus S,
\qquad W=A\otimes B,
$$

并要求 `W == Z`、`W != S`。此外，所有提交元素必须位于 `[0,256]`，`A/B` 的普通 NumPy 秩至少为 7，`P/R/S` 的普通 NumPy 秩为 8。注意这些是实数域上的 `numpy.linalg.matrix_rank`，不是热带秩。

题目来源论文 [Tropical cryptography IV](https://eprint.iacr.org/2026/095) 把安全性建立在给定尺寸的热带矩阵分解困难性上；后续论文 [A Comprehensive Break of the Tropical Matrix-Based Signature Scheme](https://eprint.iacr.org/2026/387) 指出，签名方程本身可被利用，给出了 chosen-hash 伪造、可塑性攻击和 SMT 私钥恢复。后者在“已知公钥和一份签名”的原方案模型中即可恢复完整私钥；本题没有直接给出公钥，因此官方解法利用服务允许取得的两份签名，共同约束 $X,Y$。

## 解题过程

### 官方解法：把 min-plus 方程交给 SMT

菜单选项 1 最多可调用两次。每次返回已知消息 $m_i$ 以及签名 $(A_i,B_i,P_i,R_i,S_i)$，而 $X,Y$ 在两次签名间不变，只有 $U_i,V_i$ 独立随机。

把 $X,Y,U_i,V_i$ 的每个元素声明为 `[0,255]` 内的 Z3 整数。热带乘法的一个输出元素满足

$$
c=\min(t_0,t_1,\ldots,t_{d-1}),
$$

它可精确编码为：

```python
for t in terms:
    solver.add(c <= t)
solver.add(Or(*(c == t for t in terms)))
```

第一组约束保证 $c$ 不大于所有候选值，第二组保证至少有一个候选值真正取得最小值。逐元素展开两份签名的五组等式：

$$
\begin{cases}
A_i=(M_i\otimes X)\oplus U_i,\\
B_i=(Y\otimes M_i)\oplus V_i,\\
P_i=X\otimes V_i,\\
R_i=U_i\otimes Y,\\
S_i=U_i\otimes V_i,
\end{cases}
\qquad i\in\{1,2\}.
$$

其中 $A_i$ 的逐元素约束可写成：

```python
MX = minplus_product_vars(M_i, X)
solver.add(A_i[r][c] <= MX[r][c])
solver.add(A_i[r][c] <= U_i[r][c])
solver.add(Or(A_i[r][c] == MX[r][c],
              A_i[r][c] == U_i[r][c]))
```

`B_i` 同理；`P_i/R_i/S_i` 则直接用上面的“常量等于若干项最小值”模板。求得一组与两份签名兼容的 $X,Y$ 后，选择菜单 2，读取随机的 36 字符目标消息，重新随机 $U,V$ 并按正常签名公式生成五个矩阵即可。利用流程为：

```text
取两份 (msg, A, B, P, R, S)
  -> 建立共享 X/Y、各自 U/V 的 SMT 约束
  -> 求解 X/Y
  -> 读取服务端目标消息并计算 M
  -> 随机 U/V，生成合法签名
  -> 提交 JSON
```

生成签名时还应像源码一样重试，直到 `A != U` 且 `B != V`；提交前在本地复算五个普通秩、元素范围和 min-plus 验签等式。官方利用脚本中的比赛 IP、端口只是临时靶机信息，不属于解法机制，因此不保留。

### 参赛解法：不恢复私钥，直接构造验签等式

总 PDF 还给出了一条更直接的验证器绕过。它利用 `eat` 只检查最终矩阵关系、普通秩和范围，并不验证提交的五元组是否确由某个 $U,V$ 按签名算法产生。

令

$$
r_i=\min_j M_{i,j},\qquad c_j=\min_i M_{i,j}.
$$

选择向量 $v$，使 $0\le v_j\le c_j$ 且 $r_i+v_j\le256$，定义目标矩阵

$$
Q_{i,j}=r_i+v_j.
$$

因为私钥矩阵元素非负，所以未知项满足

$$
(M\otimes T\otimes M)_{i,j}\ge r_i+c_j\ge r_i+v_j=Q_{i,j}.
$$

于是只要让其余三项都不小于 $Q$，并让每个位置至少有一项等于 $Q$，整个 `Z` 就会被压成已知矩阵 $Q$，完全不需要知道私钥。

具体构造如下。

1. 构造 $A\in\mathbb Z^{8\times7}$、$B\in\mathbb Z^{7\times8}$。令 `A[:,0]=r`、`B[0,:]=v`，其余六条中间路径全部严格大于对应的 $r_i+v_j$，便有 $A\otimes B=Q$。通过给其余列、行设置不同扰动，可同时让普通秩达到 7。
2. 对 $P$ 的第 $j$ 列，取 $t_j=\arg\min_k M_{j,k}$，设置 $P_{t_j,j}=v_j$，其余元素严格大于 $v_j$。于是 $M\otimes P\ge Q$，且对角线精确等于 $Q$。
3. 令第 $i$ 行的所有 $R_{i,k}\ge r_i$，则 $R\otimes M\ge r_i+c_j\ge Q_{i,j}$。随机扰动到 `rank(R)==8`。
4. 先令 $S=Q$，再只在已被 `M⊗P` 或 `R⊗M` 精确覆盖、且仍有数值余量的位置上增大若干元素，直到 `rank(S)==8` 且 $S\ne Q$。这样被抬高的位置仍由另一项给出 $Q$，未抬高的位置由 $S$ 给出 $Q$。

最后得到

$$
W=A\otimes B=Q=Z,qquad W\ne S,
$$

同时满足所有普通秩和 `[0,256]` 范围检查。相较 SMT 路线，这条构造利用的是本题验证器的自由度；它不代表原签名方案的一般攻击，但在本题中更短、更稳定。

成功后得到：

```text
SUCTF{W3lc0m3_t0_SU_R3stAur4nt_n3Xt_t1me!:-)}
```

## 方法总结

- 遇到自定义代数结构，先确认运算符的真实语义；本题的矩阵加法是逐元素取最小值，矩阵乘法是“加后取最小值”。
- 官方攻击把每个 min-plus 等式编码为整数 SMT 约束，用两份共享私钥的签名恢复一组可用的 $X,Y$，再按正常流程伪造。
- 验证器只检查最终等式时，还应检查能否先选定目标矩阵，再让各个 `min` 分支共同覆盖它；这可绕过对未知私钥项的求解。
- 普通矩阵秩与热带秩不同。利用代码必须按服务端实际使用的 `numpy.linalg.matrix_rank` 验证，并同时检查元素上界 256。
