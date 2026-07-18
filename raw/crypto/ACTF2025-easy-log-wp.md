# easy_log

## 题目简述

题目定义点 $P=(x,y)$ 的加法

$$
\begin{aligned}
x'&=x_0y_1+y_0x_1-x_0x_1\pmod n,\\
y'&=x_0x_1+y_0y_1\pmod n,
\end{aligned}
$$

并用二进制倍加计算标量乘法。flag 被随机前后缀填充到 118 字节后分成两段：前 68 字节转成 `flag1`，后 50 字节转成 `flag2`。

第一阶段给出固定模数 $n$、基点 $G_1$ 和 $F_1=\text{flag1}\cdot G_1$，要求恢复并提交 `flag1`；第二阶段允许选手提交一个 400 位素数 $p$，服务再返回 $F_2=\text{flag2}\cdot G_2\pmod p$。核心是识别这个自定义群的线性表示，并为两段离散对数找到可分解的群阶。

## 解题过程

### 1. 把点映射成 $2\times2$ 矩阵

定义

$$
M(x,y)=
\begin{pmatrix}
y-x & x\\
x & y
\end{pmatrix}.
$$

直接相乘可验证

$$
M(P+Q)=M(P)M(Q),
$$

所以题目的 `double_and_add(k, P, n)` 等价于计算 $M(P)^k$。自定义点群的离散对数因此可以转成矩阵乘法群中的离散对数，不必继续把它当作未知曲线研究。

矩阵行列式为

$$
N(x,y)=\det M(x,y)=y^2-xy-x^2.
$$

由行列式的乘法性可得

$$
N(P+Q)=N(P)N(Q)\pmod n.
$$

这个范数既能检查点是否可逆，也能把候选阶限制在相应代数的单位群阶中。

可用以下 Sage 骨架复现点到矩阵的转换：

```python
def point_matrix(P, modulus):
    x, y = P
    return matrix(Zmod(modulus), [[y - x, x], [x, y]])

MG = point_matrix(G, modulus)
MF = point_matrix(F, modulus)
assert MG**secret == MF
```

### 2. 恢复第一段 `flag1`

固定模数 $n$ 是合数。先分解 $n$，分别在每个素数幂模数上计算 $M(G_1)$ 的阶，再取最小公倍数得到全局阶；也可以分别求离散对数后用 CRT 合并。真正需要的是基点生成子群的阶，而不是整个矩阵群的阶。

对每个候选阶，从其素因子中逐个约去不必要因子：若

$$
M(G_1)^{r/q}=I,
$$

就令 $r\leftarrow r/q$，直到不能继续约分。题目参数使最终的 $r$ 足够光滑，可用 Pohlig–Hellman 把大离散对数拆成各个小素数幂上的离散对数，再用 CRT 合并为 `flag1`。

求得标量后必须验证：

```python
assert double_and_add(flag1, G1, n) == F1
```

服务只有在提交值与内部 `flag1` 完全相等时才进入第二阶段。

### 3. 利用 Fibonacci 结构选择第二阶段模数

第二个基点 $G_2$ 对应相邻 Fibonacci 数。若 $P_k=(F_k,F_{k+1})$，则题目的点加法满足

$$
P_a+P_b=P_{a+b},
$$

而 Cassini 恒等式给出

$$
F_{k+1}^2-F_kF_{k+1}-F_k^2=(-1)^k.
$$

这正是前述范数 $N(x,y)$。因此 $G_2$ 的阶受 Fibonacci 数列模 $p$ 的 Pisano 周期约束。

对奇素数 $p\neq5$，特征多项式 $T^2-T-1$ 的判别式为 5：

- 当 5 在 $\mathbb{F}_p$ 中是二次剩余时，特征值位于 $\mathbb{F}_p$，周期相关的阶整除以 $p-1$ 为核心的界；
- 当 5 是二次非剩余时，特征值位于 $\mathbb{F}_{p^2}$，相应阶整除以 $2(p+1)$ 为核心的界。

因此第二阶段不要随机交一个 400 位素数，而应构造使 $p-1$ 或 $p+1$ 已知且光滑的素数。得到 $F_2$ 后，先从上述可分解上界中约出 $G_2$ 的真实阶，再运行 Pohlig–Hellman：

```python
# order_bound 是依据 Legendre symbol (5/p) 选择并已知分解的上界。
order = order_bound
for q, e in factor(order_bound):
    for _ in range(e):
        if order % q == 0 and double_and_add(order // q, G2, p) == O:
            order //= q
        else:
            break

flag2 = discrete_log(F2, G2, ord=order, operation='+')
assert double_and_add(flag2, G2, p) == F2
```

实际使用 Sage 时，可直接对矩阵表示调用相应的离散对数实现，或自行按群运算实现 Pohlig–Hellman。

### 4. 拼回被随机填充的数据

恢复两段整数后按固定长度转回字节，否则前导零会丢失：

```python
blob = int(flag1).to_bytes(68, "big") + int(flag2).to_bytes(50, "big")
start = blob.index(b"ACTF{")
end = blob.index(b"}", start) + 1
print(blob[start:end])
```

随机前后缀只隐藏了 flag 在 118 字节串中的位置，并不影响两次离散对数。

## 方法总结

- 核心技巧：把自定义二元运算识别为 $2\times2$ 矩阵乘法，再在可控阶的子群中求离散对数。
- 识别信号：运算式对两个点的坐标均为双线性形式，且出现 $y^2-xy-x^2$ 或相邻 Fibonacci 数，应检查矩阵表示、范数和 Pisano 周期。
- 复用要点：第二阶段能自选模数时，应主动构造 $p-1$ 或 $p+1$ 光滑的素数；求解前要约出基点真实阶，恢复字节时要保留题目规定的固定长度。
