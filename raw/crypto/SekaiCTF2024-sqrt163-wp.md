# √163

## 题目简述

题目实现了一个 CSIDH 风格的类群作用。小素数集合为 $3\leq\ell<128$，并额外加入 $163$，有限域素数为：

$$
p=4\prod_{\ell}\ell-1.
$$

服务先计算只使用一次 $163$-isogeny 的公开曲线 `pub_163`，再要求玩家提交一个短整数向量 $\mathbf v$，满足：

$$
\operatorname{csidh}(\operatorname{csidh}(E_0,\mathbf v),\mathbf v)
=\operatorname{csidh}(E_0,\mathbf e_{163}).
$$

也就是说，要在 CSIDH 类群中求出 $163$-isogeny 类的“平方根”。成功后，以中间曲线参数的 SHA-256 作为 AES-ECB 密钥解密 flag。

## 解题过程

### 把 isogeny action 转成理想类群运算

CSIDH 的交换作用可用虚二次序的理想类群描述：每个小素数 $\ell$ 对应判别式 $-4p$ 下的一个理想类，连续执行 isogeny 等价于在类群中相乘。这个对应关系也是题解引用的 [CSI-FiSh 论文](https://eprint.iacr.org/2019/498) 中用于高效类群计算的基础；本文所需信息是“isogeny 指数向量可转成类群元素，组合 action 对应类群加法”，无需依赖论文完成后续步骤。

PARI 可以直接计算该二次序的类群：

```python
ells = [*primes(3, 128), 163]
p = 4 * prod(ells) - 1
pari.quadclassunit(-4 * p)
```

得到类群为循环群，阶为：

```text
102019419125180345266808265
```

其分解为：

$$
3^2\cdot5\cdot13^2\cdot14153\cdot130241\cdot7277586888541,
$$

整体较平滑。用二元二次型：

$$
Q_\ell=(\ell,2,(p+1)/\ell)
$$

表示每个 $\ell$-isogeny 对应的类。$Q_3$ 恰好生成整个循环群，因此可把所有 $Q_\ell$ 都取相对于 $Q_3$ 的离散对数：

```python
order = 102019419125180345266808265
q3 = pari.Qfb(3, 2, (p + 1) // 3)

dlogs = []
for ell in ells[1:]:
    qell = pari.Qfb(ell, 2, (p + 1) // ell)
    dlogs.append(discrete_log(
        qell, q3, order,
        operation=None,
        identity=q3**0,
        inverse=lambda x: x**-1,
        op=lambda a, b: a * b,
    ))
```

### 求短的“半个 163 类”表示

类群阶为奇数，所以 $2$ 在模 `order` 下可逆。目标变成寻找短向量 $\mathbf v$，使其离散对数满足：

$$
\sum_i v_i\log_{Q_3}(Q_{\ell_i})
\equiv \frac{1}{2}\log_{Q_3}(Q_{163})pmod{\lvert\mathrm{Cl}\rvert}.
$$

把模关系、各坐标单位向量和目标 $1/2$ 放入 Kannan embedding，再用 LLL 找短表示：

```python
M = matrix(32)
M[0, 0] = order
M[1:-1, 1:-1] = identity_matrix(30)
M[1:-1, 0] = -vector(dlogs)
M[-1, -2] = mod(1/2, order)
M[-1, -1] = 2**1024

private_key = tuple(M.LLL()[-1][:-1])
```

得到可直接提交的短向量：

```text
(2, -4, 1, 0, -1, -2, -2, -2, -2, 0, -1, -1, -4, 0, -1,
 4, 1, -1, 2, -4, 0, 2, -7, -5, 1, 0, 2, 1, -1, 0, 0)
```

服务验证两次执行该 action 等于一次 $163$-isogeny action，随后解密得到：

```text
SEKAI{isogenni_where?_its_all_ideal_classes!}
```

README 中的视频链接实际是题目主题曲 `Sqrt(N) - Capture / feat.初音ミク`，不包含解题所需技术信息，因此正文不保留为必要引用。

## 方法总结

- 核心技巧：把 CSIDH isogeny 向量映射到虚二次序的理想类群，在循环群中做离散对数，再用 LLL 寻找目标类的短整数表示。
- 识别信号：要求寻找某个 isogeny action 的平方根或组合根，且参数允许计算判别式对应的类群结构。
- 复用要点：先验证类群是否循环、生成元阶数和群阶分解；数学上求得模类群的解后，还必须寻找系数足够短的等价向量，才能让具体 CSIDH 实现可在时限内执行。
