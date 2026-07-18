# AAALLL

## 题目简述

题目在环

$$
R=\mathbb{F}_p[X]/(X^n+1),\qquad n=450,\quad p=3774877201
$$

中随机生成系数属于 $\{-1,0,1\}$ 的三元多项式 $f$。程序从 $X^n+1$ 在 $\mathbb{F}_p$ 中的全部根里随机选取 $t=n/2=225$ 个根，公开这些根组成的 `subset` 及对应的 $f(x_i)$。最后使用

```python
key = md5(str(f.list()).encode()).digest()
ct = AES.new(key, AES.MODE_ECB).encrypt(pad(flag, 16))
```

加密 flag。因此目标不是直接攻击 AES，而是从部分 Vandermonde 取值恢复三元秘密 $f$。

这是 Partial Vandermonde Knapsack 的密钥恢复实例。Das 与 Joux 的论文 [Key Recovery Attack on the Partial Vandermonde Knapsack Problem](https://eprint.iacr.org/2024/366) 给出了利用代数结构降维后再做格规约的思路；本题参数较弱，降维后的两个实例可直接使用 primal lattice attack 和 LLL 求解，不必复现论文中更复杂的参数处理。

## 解题过程

### 1. 在公开根中寻找逆元对

若 $x^n=-1$，则 $(x^{-1})^n=-1$，所以根集合关于求逆封闭。`subset` 随机取全部根的一半，期望其中约一半元素能够在子集中找到自己的逆元。将这些元素配成

$$
(x,x^{-1}),\qquad xx^{-1}\equiv 1\pmod p
$$

并同步取得公开值 $(f(x),f(x^{-1}))$。

在 $R$ 中有 $X^n=-1$，因而

$$
X^{-1}=-X^{n-1}.
$$

设

$$
f(X)=f_0+f_1X+\cdots+f_{n-1}X^{n-1}.
$$

对一对互逆根分别求和、求差，可得

$$
\begin{aligned}
f(x)+f(x^{-1})
&=2f_0+\sum_{i=1}^{n/2-1}(f_i-f_{n-i})(x^i-x^{n-i}),\\
f(x)-f(x^{-1})
&=\sum_{i=1}^{n/2-1}(f_i+f_{n-i})(x^i+x^{n-i})
  +2f_{n/2}x^{n/2}.
\end{aligned}
$$

原本 $n$ 维的秘密被拆成两个约 $n/2$ 维的秘密向量，其系数只可能属于 $\{-2,-1,0,1,2\}$，仍然是格中的短向量。

### 2. 分别建立两个短向量实例

对每个逆元对 $(x_j,x_j^{-1})$，使用下列行向量建立模 $p$ 线性系统：

```python
# 求和系统，对应 f(x) + f(x^-1)
row_plus = [2]
row_plus += [
    (xj**i - xj**(n-i)) % p
    for i in range(1, n // 2)
]

# 求差系统，对应 f(x) - f(x^-1)
row_minus = [
    (xj**i + xj**(n-i)) % p
    for i in range(1, n // 2 + 1)
]
```

右端分别为

$$
b_j^+=f(x_j)+f(x_j^{-1}),\qquad
b_j^-=f(x_j)-f(x_j^{-1})\pmod p.
$$

将右端拼到矩阵最后一列，求其右核，再把模 $p$ 的核提升到整数格并补上 $pI$：

```python
def recover_short(W, b, p):
    Wb = block_matrix([W, b], ncols=2)
    kernel = Wb.right_kernel().matrix().change_ring(ZZ)
    lattice = kernel.stack(identity_matrix(Wb.ncols()) * p)
    reduced = lattice.LLL()

    # 选择系数落在 {-2,-1,0,1,2} 且能通过原方程验证的短行。
    for row in reduced:
        coeffs = list(row[:-1])
        if max(abs(int(v)) for v in coeffs) <= 2:
            return row
    raise ValueError("short vector not found")
```

最后一维是齐次化时引入的坐标，应在验证符号和模方程后移除。不要只按向量长度盲选：候选还必须满足公开的取值方程和三元系数约束。

### 3. 合并两组结果并解密

设求和系统恢复出

$$
a_i=f_i-f_{n-i},
$$

求差系统恢复出

$$
b_i=f_i+f_{n-i}.
$$

则

$$
f_i=\frac{a_i+b_i}{2},\qquad
f_{n-i}=\frac{b_i-a_i}{2}\pmod p.
$$

结合单独恢复的 $f_0$ 与 $f_{n/2}$，按原顺序重建全部 450 个系数，再把模 $p$ 表示规范化为 `-1/0/1`。最后复现题目中的列表序列化方式生成 MD5 密钥并解密：

```python
coeffs = [int(v) for v in recovered_f]
assert set(coeffs) <= {-1, 0, 1}

key = md5(str(coeffs).encode()).digest()
plain = AES.new(key, AES.MODE_ECB).decrypt(ct)
print(unpad(plain, 16))
```

## 方法总结

- 核心技巧：利用 $X^n+1$ 根集合的逆元对，把 Partial Vandermonde 实例拆成两个半维短向量问题，再用核格与 LLL 恢复秘密。
- 识别信号：公开三元多项式在单位根或 negacyclic 环根上的部分取值，同时参数满足 $X^n=-1$，应优先检查根集合的共轭、逆元或自同构配对。
- 复用要点：格规约只负责产生短候选；必须再用公开方程、系数范围和原始顺序验证，最后严格复现题目对系数列表的序列化方式。
