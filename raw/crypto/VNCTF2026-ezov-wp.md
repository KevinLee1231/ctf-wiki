# ezov

## 题目简述

题目是 Oil-and-Vinegar / UOV 风格签名伪造。附件 `main.sage` 中给出公钥 `pub.txt`，服务端允许对任意非 `admin` 消息签名，但验证阶段只要提交 `admin` 的合法签名就返回 flag。私钥中心映射的矩阵具有典型 OV 结构，公开矩阵为：

```python
Q = block_matrix(F, [[Qvv, Qvo], [Qvo.transpose(), 0]])
P = T.transpose() * Q * T
```

其中 oil-oil 块为 0。签名逻辑是先固定 vinegar 变量，再把 oil 变量作为线性方程求解：

```python
def sign(self, msg):
    M = self.Hash(msg.encode())
    while True:
        V = random_vector(self.F, v).list()
        Y = vector(PR, V + list(X_oil))
        ...
        if L.is_invertible():
            X_oil = L.solve_right(target)
            Y = vector(self.F, V + list(X_oil))
            return T.inverse() * Y
```

服务端限制点也很直接：

```python
if choice == 1:
    msg = input("your msg:")
    if msg == "admin": raise RuntimeError
    sig = ov.sign(msg)

if choice == 2:
    sig = ...
    if ov.verify("admin", sig): print(flag)
```

因此目标不是从签名 oracle 直接签 `admin`，而是从公开二次型恢复足够的中心结构，自己伪造 `admin` 签名。

## 解题过程

题目参数为：

```python
o, v = 64, 64
n = o + v
q = 0x10001
F = GF(q)
```

`pub.txt` 中保存 64 个 `128 x 128` 的公开二次型矩阵。矩阵很大，不适合全文放进 WP；只需要知道它们来自 `P_i = T^T Q_i T`，且中心矩阵 `Q_i` 的 oil-oil 块为 0。

官方题解参考了 [这篇文章](https://zhuanlan.zhihu.com/p/440168430) 的思路。先随机取若干公开矩阵的线性组合：

```python
pub = read_pub("pub.txt", o=o, n=n, q=q)
Ms = []
for _ in range(v):
    c = vector(F, [F.random_element() for _ in range(o)])
    M = sum(c[i] * pub[i] for i in range(o))
    Ms.append(M)
```

对两个组合矩阵构造：

```python
W12 = Ms[0]^(-1) * Ms[1]
ws = W12.characteristic_polynomial()
h = sqrt(ws)
K = h(W12).right_kernel_matrix()
```

这里利用的是 OV 结构带来的不变量。通过矩阵 pencil 的核可以恢复一个等价的 vinegar/oil 子空间变换。之后构造等价变换矩阵：

```python
T = K.stack(Matrix(F, v, v).augment(identity_matrix(F, v)))
Fs = [T * P * T.transpose() for P in pub]
```

把每个二次型矩阵转成多项式：

```python
R = PolynomialRing(F, "x", n)
x = R.gens()

def F2f(Fx):
    f = 0
    for i in range(n):
        f += Fx[i, i] * x[i]^2
        for j in range(i + 1, n):
            f += 2 * Fx[i, j] * x[i] * x[j]
    return f

fs = [F2f(P) for P in Fs]
```

然后对目标消息 `admin` 计算 hash：

```python
from hashlib import shake_128

def Hash(msg):
    h = shake_128(msg.encode()).hexdigest(128)
    return vector(F, [int(h[i:i+4], 16) for i in range(0, len(h), 4)])

hs = Hash("admin")
```

伪造阶段仍沿用 OV 的正常签名方式：随机给出 vinegar 变量，剩余 oil 变量满足线性方程。官方 exp 中用 Gröbner basis 求出一组解：

```python
vs = [F.random_element() for _ in range(v)]
vars = list(R.gens()[:v])
args = vars + vs

ffs = [fs[i](*args) - hs[i] for i in range(o)]
I = ideal(ffs)
gb = I.groebner_basis()

solution_dict = {}
for poly in list(gb):
    var = poly.variables()[0]
    const = poly.constant_coefficient()
    solution_dict[var] = -const

sols = [solution_dict[xi] for xi in x[:len(solution_dict)]] + vs
y = vector(F, sols)
x_sig = T.transpose() * y
```

最后本地调用 `verify("admin", x_sig)` 验证通过，即可提交签名。

## 方法总结

- 核心技巧：UOV/OV 结构恢复与签名伪造。
- 识别信号：公开多组二次型矩阵，中心结构存在 oil-oil 零块，服务端禁止签某个目标消息但允许验证该目标。
- 复用要点：先从公开矩阵的线性组合恢复等价的 oil/vinegar 子空间，再按 OV 正常签名流程固定 vinegar、求解 oil。大型 `pub.txt` 属于挑战数据，WP 中保留参数和矩阵结构即可，不需要全文贴出。
