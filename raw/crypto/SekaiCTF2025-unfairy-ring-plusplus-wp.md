# unfairy-ring++

## 题目简述

题目固定六把 UOV 公钥：

```python
names = ['Miku', 'Ichika', 'Minori', 'Kohane', 'Tsukasa', 'Kanade']
pks = [uov.expand_pk(uov.shake256(name.encode(), 43576)) for name in names]
```

提交内容被拆成六个 112 字节块 $\mathbf{x}_0,\ldots,\mathbf{x}_5$，验证器要求

$$
\bigoplus_{j=0}^{5}F_j(\mathbf{x}_j)
=\operatorname{SHAKE256}(\texttt{SEKAI},44).
$$

每个 $F_j:\mathrm{GF}(256)^{112}\rightarrow\mathrm{GF}(256)^{44}$ 都是齐次二次映射。因此本题等价于在 $6\times112=672$ 个域元素上求解 44 个二次方程。

它与 `unfairy-ring` 的唯一区别看似只是删除第七把 `Mai` 公钥，实际却使原有降维日程无法闭合。沿用七公钥解法需要猜测五个方程，复杂度约为

$$
256^5=2^{40},
$$

这已经不适合两天赛程。完整解法必须再利用半线性变量和齐次缩放，把复杂度压到约 $2^{24}$。

题目来源仍是 DiceCTF 2025 [fairy-ring](https://github.com/defund/ctf/tree/master/dicectf-quals-2025/fairy-ring)，但原题依赖重复公钥的线性化攻击；当前验证器把六个不同公钥写死，不能照搬原脚本。

## 解题过程

### 1. 建立块正交 MQ 系统

将 UOV 公钥展开后，第 $i$ 个输出分量可写成

$$
\sum_{j=0}^{5}\mathbf{x}_j^{\mathsf T}A_{i,j}\mathbf{x}_j=t_i,
\qquad 0\leq i<44.
$$

矩阵 $A_{i,j}$ 的大小为 $112\times112$。不同公钥块之间没有乘积项，这是后续算法能够工作的重要条件。

对二次型

$$
f_A(\mathbf{x})=\mathbf{x}^{\mathsf T}A\mathbf{x}
$$

定义极化形式

$$
f'_A(\mathbf{x},\mathbf{y})
=\mathbf{x}^{\mathsf T}(A+A^{\mathsf T})\mathbf{y}.
$$

它在特征 $2$ 上是双线性的。通过为某个变量块构造线性变换

$$
\mathbf{x}=T\mathbf{y},
\qquad
T=(\mathbf{t}_0,\mathbf{t}_1,\ldots),
$$

并令其余列落在

$$
\mathbf{t}_0^{\mathsf T}(A_i+A_i^{\mathsf T})\mathbf{t}_r=0
$$

的公共核内，可让新变量 $y_0$ 不再与其余变量产生二次交叉项。方程行变换随后把包含 $y_0$ 的部分集中到一个方程，递归求解其余方程，最后再反向恢复 $y_0$。

### 2. 从纯平方变量扩展到半线性变量

七公钥版本先把单变量整理成 $y_0^2$。不做秩优化时，一轮普通降维会从当前变量块消耗 $m+1$ 个维度，即

$$
\operatorname{MQ}^{m}(n_1,\ldots,n_k)
\longrightarrow
\operatorname{MQ}^{m-1}(n_1-m-1,\ldots,n_k).
$$

利用极化矩阵奇异线性组合降低核条件的秩后，通常可把损失压到 $m$ 个维度。六公钥版本还要进一步允许

$$
a_i\bigl(y_0^2+y_0\bigr)+Q_i(\mathbf{y}_{\mathrm{rest}}).
$$

$y_0$ 的一次项不再为零，但它在各方程中与 $y_0^2$ 的系数保持同一比例，因此仍可通过方程行变换同时消去。半线性条件再省去一个维度，使降维进一步改进为

$$
\operatorname{MQ}^{m}(n_1,\ldots,n_k)
\longrightarrow
\operatorname{MQ}^{m-1}(n_1-m+1,\ldots,n_k).
$$

代价是反向恢复不再只是开平方，而要解

$$
y_0^2+u y_0+c=0.
$$

在 $\mathrm{GF}(2^8)$ 中，这类方程可能没有根，也可能有两个根。实现不能只保留一个返回值，而应让每一层接收一组候选并输出所有分支：

```text
solutions = terminal_solutions
for transform in reversed(path):
    next_solutions = []
    for solution in solutions:
        roots = solve_x2_plus_u_x_plus_c(solution)
        for root in roots:
            next_solutions.append(transform.backward(solution, root))
    solutions = next_solutions
```

单次反向步骤有约一半概率无根、另一半概率出现两个根，所以平均分支数仍约为一。作者代码预计算所有 $(u,c)$ 对应的根表，并通过 `batch_compress_backward` 批量传播分支。

半线性变换的构造条件比纯平方版本更严格。除极化矩阵的核条件外，还要让相应二次型在选中的方向上满足所需的零值关系；无法满足时就退回普通线性降维。官方日程中特别对编号 2、4 的变量块以及方程数很小时使用普通版本。

### 3. 用齐次性免费满足一个方程

只靠半线性降维仍需猜四个方程，即约 $2^{32}$ 次筛选。UOV 公钥映射是齐次二次映射：

$$
F(a\mathbf{x})=a^2F(\mathbf{x}).
$$

先对 44 个方程做可逆行变换，挑出一个右端非零的方程并将它归一化，再用它消去其余 43 个方程的常数项。这样剩余 43 个方程全部变成右端为零的齐次方程。

若 $\mathbf{x}$ 满足这 43 个零方程，则任意 $a\mathbf{x}$ 仍满足它们。设被单独留下的归一化方程在候选上的二次值为 $r\neq0$，目标值为 $v$，只需取

$$
a=\sqrt{\frac{v}{r}},
$$

整体缩放六个签名块，就能同时保持 43 个零方程并满足最后一个非零方程。因为 $\mathrm{GF}(2^8)$ 上平方映射可逆，这个缩放量总能求出。该方程因此不需要以 $1/256$ 的概率碰撞，等价于得到一个“免费方程”。

43 个齐次零方程中取 40 个进入降维系统，余下三个用于最终筛选。复杂度降为

$$
256^3=2^{24}.
$$

### 4. 六块变量的降维日程

作者给出的块选择顺序为：

```python
SEQ = [
    4, 5, 5, 5, 4, 4, 4, 2, 3, 2, 3, 3,
    3, 2, 2, 1, 1, 1, 1, 1, 1, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
]
```

求解器的完整流程如下：

```text
1. 从六把 UOV 公钥还原 44 组二次型矩阵。
2. 对方程做可逆随机行变换，选出一个常数项非零的方程 last_eq。
3. 用 last_eq 把其余 43 个方程变成常数项为零的齐次方程。
4. 取前 40 个方程，按 SEQ 交替执行普通降维和半线性降维。
5. 在最终 5 维空间中随机选择后四个坐标，解首坐标的二次方程。
6. 逆序执行全部变换，并保留每层产生的 0 个或 2 个分支。
7. 检查未参与降维的三个齐次方程。
8. 计算 last_eq 的当前值 r，并以 sqrt(v/r) 整体缩放。
9. 用原始 44 个方程以及 uov.pubmap 各验证一次。
10. 将六个 112 字节块拼接成十六进制字符串后提交。
```

与七公钥版本一样，应提前把各层二次型压缩到末端小空间，否则约 $2^{24}$ 次尝试中的矩阵计算会成为瓶颈。半线性版本还要缓存有限域二次方程的根，并批量处理分支。

完整 Sage 实现及每一种 `Transform` 的正向、压缩和反向逻辑见[出题人解法](https://sekai.team/blog/sekaictf-2025/unfairy-ring)。本地源码中的 `SEKAI{TESTFLAG}` 只是环境变量缺失时使用的测试占位符，不是比赛 flag。

## 方法总结

`unfairy-ring++` 说明少一个变量块会让欠定 MQ 的复杂度发生质变。普通正交变量只能把待恢复部分化为平方方程；半线性变量允许用更少的维度完成同一轮消元，但反向过程必须正确处理 $x^2+ux+c=0$ 的零根或双根分支。

最后一个关键点是齐次性。先把目标向量通过方程行变换集中到一个坐标，求出其余齐次零方程的解，再整体缩放匹配该坐标，相当于省去一次 $1/256$ 的随机检查。最终复杂度由五方程猜测的 $2^{40}$ 降到三方程猜测的 $2^{24}$。任何候选都必须回到原始公钥映射进行全量复核，不能把降维系统或压缩系统的通过当作最终正确性证明。
