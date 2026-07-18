# tinyCKKS

## 题目简述

题目实现了一个简化 CKKS，工作在

$$
R_q=\mathbb{Z}_q[X]/(X^N+1),\qquad
N=1024,\quad p=947819,\quad q=p^5,
$$

并取缩放因子 $\Delta=p$。长度为 $N$ 的浮点向量按系数编码为

$$
\operatorname{Encode}(m)_i=\operatorname{round}(\Delta m_i).
$$

秘密 $s$ 是三元多项式。加密随机选择 $a$ 和小误差 $e$，输出

$$
(a,b),\qquad b=\operatorname{Encode}(m)+e-as\pmod q.
$$

解密得到

$$
b+as=\operatorname{Encode}(m)+e,
$$

再逐系数除以 $\Delta$ 返回近似浮点数。服务共进行七轮：可请求同态操作并提交结果让服务解密、直接查看原始明文，或提交精确明文多项式；最后一种只有完全相等时才给 flag。

漏洞由两部分组成：近似解密结果保留了误差低位，可用于恢复秘密 $s$；误差又由 Python `random.getrandbits(4)` 生成，收集足够多的 4 位输出后可以恢复 MT19937 状态并预测下一轮误差。

## 解题过程

### 1. 利用近似解密结果恢复秘密多项式

第一轮选择加法操作。服务另给一份密文，目标明文变成两份编码明文之和；把两份密文相加后原样提交，服务会解密并返回 `new_plain`。

由于误差系数远小于 $\Delta$，将返回值乘以 $p$ 并四舍五入，可以精确恢复解密多项式

$$
u=\operatorname{round}(p\cdot\texttt{new\_plain})=b+as.
$$

于是

$$
as=u-b\pmod q.
$$

环乘法是模 $X^N+1$ 的负循环卷积。把乘以 $a$ 写成 $N\times N$ 矩阵即可解出 $s$：

```python
def negacyclic_matrix(a):
    q, N = a.q, a.N
    coeffs = a.poly.list()
    M = matrix(Zmod(q), N, N)
    for i in range(N):
        for j in range(N):
            M[i, (i + j) % N] = coeffs[j] if i + j < N else -coeffs[j]
    return M

def recover_sk(a, rhs):
    M = negacyclic_matrix(a)
    v = vector(Zmod(a.q), rhs.poly.list())
    coeffs = list(M.solve_left(v))
    return polynomial(a.q, a.N, coeffs)

decoded_poly = polynomial(q, N, [round(p * x) for x in new_plain])
sk = recover_sk(ct.a, decoded_poly - ct.b)
assert ct.b + ct.a * sk == decoded_poly
```

这里与 BFV 的差别非常关键：BFV 的最终舍入会主动丢掉噪声，而本题的 CKKS 风格解码把 $(m+e/\Delta)$ 作为浮点数返回。只要输出精度足以在乘回 $\Delta$ 后正确舍入，误差就没有被隐藏。

### 2. 从已知明文轮次提取误差

后续五轮选择 `c = 2`，服务会打印当前原始浮点明文。重新执行编码即可得到精确 $pt$：

```python
pt = polynomial(q, N, [round(p * x) for x in plain])
error = ct.b + ct.a * sk - pt
```

误差生成函数为：

```python
def sample_gaussian_poly(Q):
    half = Q.degree() // 2
    return Q(
        [getrandbits(4) for _ in range(half)]
        + [-getrandbits(4) for _ in range(half)]
    )
```

因此先把模 $q$ 的负数恢复成绝对值，就能按调用顺序得到每个 `getrandbits(4)` 的返回值：

```python
observed = []
for c in error.poly.list():
    v = int(c)
    observed.append(v if v < 16 else q - v)
```

每轮提供 1024 个 4-bit 观测，五轮共有 5120 个观测，即 20480 位，超过 MT19937 的 624 个 32-bit 状态字所含的 19968 位。需要使用支持“已知每个 32 位输出最高 4 位”的状态恢复实现，而不是把 4 位值简单拼接成 32 位整数。

恢复后先逐个重放已有观测，确认克隆状态和调用位置都正确：

```python
rng = MT19937(state_from_data=(observed, 4))
for value in observed:
    assert value == rng() >> 28
```

### 3. 预测第七轮误差并提交精确明文

下一轮加密仍按“前半正、后半负”的顺序取 1024 个 4 位数：

```python
half = N // 2
predicted = [rng() >> 28 for _ in range(half)]
predicted += [-(rng() >> 28) for _ in range(half)]
e = polynomial(q, N, predicted)
```

收到第七轮密文 $(a,b)$ 后，直接计算

$$
pt=b+as-e\pmod q.
$$

选择 `c = 3` 并按服务要求的字符串格式提交该多项式，服务端比较 `your_plain == target_pt`，完全一致后输出 flag：

```python
exact_pt = ct.b + ct.a * sk - e
send_polynomial(exact_pt)
```

另有一条非预期路线：`sample_uniform_poly` 的符号由 NumPy 随机数生成器选择，而明文也来自 `np.random.uniform`。密文中的 $a$ 会泄露符号选择结果，因此可以恢复 NumPy 的 MT19937 状态并预测后续明文。这条路线绕过了秘密多项式恢复，但预期解法更直接地利用了 CKKS 近似解码与 Python `random` 的 4 位误差。

## 方法总结

- 核心技巧：从 CKKS 近似解密值乘回缩放因子，恢复含噪声的精确环元素，进而解线性方程得到秘密；再用多轮误差恢复 MT19937 状态。
- 识别信号：服务返回高精度近似明文、误差远小于缩放因子，并重复使用可预测 PRNG 生成小误差时，应检查是否能恢复 $pt+e$ 和 PRNG 输出。
- 复用要点：负循环卷积矩阵的符号必须与 $X^N+1$ 一致；恢复 PRNG 后先重放全部观测校准状态，再预测下一轮，避免因一次额外随机调用导致整体错位。
