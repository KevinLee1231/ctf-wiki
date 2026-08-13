# PRG

## 题目简述

服务端连续进行 100 轮判别游戏，每轮给出 16 字节真随机数据或自制 PRG 输出，必须全部判断正确。PRG 使用三个 64 位未知向量 $x,r,k$ 和公开的 $64\times64$ 二元矩阵 $A$；每次输出 $x$ 的位奇偶校验，再按轮次对 $x$ 做线性更新。整个生成器因此只是 $\operatorname{GF}(2)$ 上的线性系统。

## 解题过程

把初始状态 $x_0$、常量 $r$、$k$ 的 192 个比特设为符号变量。每个输出位为

$$
o_i=\sum_{j=0}^{63}x_{i,j}\pmod2,
$$

状态更新依次循环：

$$
x_{i+1}=Ax_i+r,\quad Ax_i+k,\quad Ax_i+r+k.
$$

这些表达式始终是 192 个初始变量的线性组合。将收到的前 80 位代入，就得到一个二元线性方程组。若输出来自 PRG，方程组必然有解；若来自真随机源，则很可能违反生成器固有的线性关系。

官方脚本用 Sage 多项式环表达同一个一致性检查：

```python
F = PolynomialRing(GF(2), 192, "x")
variables = F.gens()
x = vector(F, variables[:64])
r = vector(F, variables[64:128])
k = vector(F, variables[128:])

equations = []
for i, bit in enumerate(output_bits[:80]):
    equations.append(sum(x) - bit)
    if i % 3 == 0:
        x = A * x + r
    elif i % 3 == 1:
        x = A * x + k
    else:
        x = A * x + r + k

basis = Ideal(equations).groebner_basis()
guess = 0 if basis == [F(1)] else 1
```

`[1]` 表示理想包含常数 1，即方程不相容，应判断为真随机；否则判断为 PRG。对题目给定矩阵直接做二元高斯消元可验证：前 80 个输出方程的系数矩阵秩只有 63，存在 17 条独立线性约束，所以随机 80 位误通过的概率约为 $2^{-17}$。逐轮执行即可取得：

```text
grey{Not_so_easy_to_construct_a_secure_PRG_LaQSqprzmTjBZs8ygMkGuw}
```

## 方法总结

线性状态更新配上线性输出函数不会产生密码学安全的伪随机性，即使内部名义上有 192 个未知比特。判断这类生成器不必恢复状态，只需检查观察值是否落在输出线性映射的像空间中。实现安全 PRG 时必须引入经过分析的非线性构造，而不能把公开矩阵运算和固定向量相加误当作不可预测性。
