# SUCTF2026-RSA

## 题目简述
题目是 RSA 部分信息泄露。附件 `SU_RSA.py` 的价值在于确认参数生成方式：`d` 偏小，同时 `p+q` 的高位已知、低位未知；真正解法是把 `ed = kφ(N)+1` 与 `φ(N)=N-(p+q)+1` 结合，构造关于小未知量的 Coppersmith/Boneh-Durfee 问题。密文和大整数只作为输入数据，不需要在简述中全文展开。

## 解题过程
这是一道非常经典的基于 Coppersmith 方法和 Boneh-Durfee 攻击的 RSA 部分密钥泄露（Partial
Key Exposure）题目。

从给定的代码中可以提取出以下关键信息：

1. `d` 较小：`d` 的长度约为 `1024 * 0.33 ≈ 337` bits。

2. 部分 `p + q` 已知：$S$ 保留了 `p + q` 的高位，将低约 399 bits 清零。也就是说 $p+q=S+x$，其中 $x < 2^{399}$。

3. 根据 RSA 的原理，存在整数 $k$ 使得：

$$
e d = k \varphi(N) + 1
$$

4. 代入 $\varphi(N)=N-(p+q)+1=N-S-x+1$。

5. 令已知常量 $A=N-S+1$，方程变为：

$$
e d = k(A-x)+1
$$

6. 两边对 $e$ 取模，得到二元同余方程：

$$
k(A-x)+1 \equiv 0 \pmod e
$$

由于 $k < 2^{337}$ 且 $x < 2^{399}$，它们的乘积约小于 $2^{736}$，远小于 $e \approx 2^{1024}$。这符合 Boneh-Durfee 攻击，也可以视作二维 Coppersmith 小根问题。

我们可以通过构建格（Lattice）并使用 LLL 算法来求出小根 x 和 $k$，进而还原 ϕ(N) 并解出 d

为了使用 LLL 算法规约，我们需要构建一个方阵。原脚本尝试把 33 个多项式填入 63 行的矩阵中，当
循环到第 34 行（即 i=33 ）时，polys[i] 就越界了，直接触发了 IndexError: list
index out of range 。

导致单项式数量“爆炸”的原因在于，在这个特制的方程 $f(k,x)=kx-Ak-1$ 中，$x$ 是从
不单独出现的（它总是和 k 绑定在一起）。原脚本错误地用 xj 进行了“常规位移”，导致生成了大
量无法互相抵消的高次项。

我们需要：

1. 修正循环边界：常规位移（k-shifts）的边界应该是 m - i + 1 ，而不是 i + 1 。

2. 对调位移变量：用 kj 进行常规位移，用 xj 进行扩展位移，这样生成的多项式数量和提取出的单
项式数量就会完美匹配（例如 m = 5, t = 3 时，都是 39 个）。

3. 优化求根逻辑：`ideal.variety()` 在某些版本的 Sage 中处理整数环上的多元方程组不稳定，因此改用 CTF 中更稳的结式（Resultant）消元法。

```python
from sage.all import *
from Crypto.Util.number import long_to_bytes

N =

e =

c =

S =

bits = 1024
delta0 = 0.33
gamma = 0.39

A = N - S + 1
X_bound = int(2**(bits * gamma)) # x 的上限边界
K_bound = int(2**(bits * delta0)) # k 的上限边界

# 修正：适当调整 m 和 t 以保证方阵及规约精度
m_val = 5
t_val = 3

PR = PolynomialRing(ZZ, names=('k', 'x'))
k, x = PR.gens()
f = k*x - A*k - 1

print("[*] 正在构造位移多项式...")

polys = []

# 1. 修正的常规位移：使用 k，且边界为 m_val - i + 1
for i in range(m_val + 1):
for j in range(m_val - i + 1):
polys.append((k**j) * (f**i) * (e**(m_val - i)))

# 2. 修正的扩展位移：使用 x
for i in range(m_val + 1):
for j in range(1, t_val + 1):
polys.append((x**j) * (f**i) * (e**(m_val - i)))

# 提取并排序所有的单项式
monomials = set()
for p in polys:
monomials.update(p.monomials())
monomials = sorted(list(monomials))
dim = len(monomials)

print(f"[*] 多项式数量: {len(polys)}")
print(f"[*] 单项式数量: {dim}")
print(f"[*] 格的维度: {dim} x {dim}")

if len(polys) != dim:
print("[-] 错误：多项式数量与单项式数量不一致，无法构造方阵！请检查位移逻辑。")
exit()

# 构建格的基矩阵
print("[*] 正在构建矩阵...")
M = Matrix(ZZ, dim, dim)
for i in range(dim):
p = polys[i]
for j in range(dim):
mon = monomials[j]
coeff = p.monomial_coefficient(mon)
M[i, j] = coeff * mon(K_bound, X_bound)

print("[*] 正在执行 LLL 规约 (通常几秒到十几秒完成)...")
M_LLL = M.LLL()

print("[*] 正在重构多项式...")
roots_polys = []
for i in range(dim):
p_lll = 0
for j in range(dim):
coeff = M_LLL[i, j] // monomials[j](K_bound, X_bound)
p_lll += coeff * monomials[j]

roots_polys.append(p_lll)

print("[*] 正在通过 Resultant (结式) 提取根...")
try:
# 切换到有理数域 QQ，求结式更稳定
PR_QQ = PolynomialRing(QQ, names=('k', 'x'))
k_qq, x_qq = PR_QQ.gens()

p1 = PR_QQ(roots_polys[0])
found = False

# 防止多项式非代数独立，尝试前几个短向量
for p_idx in range(1, 4):
p2 = PR_QQ(roots_polys[p_idx])
res = p1.resultant(p2, k_qq) # 消去 k，得到只含 x 的多项式

if res.is_zero():
continue

res_roots = res.univariate_polynomial().roots()
for x_val, _ in res_roots:
if x_val.is_integer():
x_val = int(x_val)
print(f"\n[+] 成功找到未知量 x: {x_val}")

# 还原真实参数并解密
phi = A - x_val
d = int(inverse_mod(e, phi))
m_pt = int(pow(c, d, N))

flag = long_to_bytes(m_pt)
print(f"[+] FLAG: {flag.decode('utf-8', errors='ignore')}")
found = True
break
if found:
break

if not found:
print("[-] LLL 成功，但未能在整数域内找到对应的 x 根。")

except Exception as err:
print("[-] 求解过程出现异常:", err)
```

脚本运行后能够构造 `39 x 39` 的格，并通过 Resultant 找到未知低位 `x`，随后还原 `phi` 和私钥：

```text
[*] 正在构造位移多项式...
[*] 多项式数量: 39
[*] 单项式数量: 39
[*] 格的维度: 39 x 39
[*] 正在执行 LLL 规约...
[*] 正在通过 Resultant 提取根...
[+] 成功找到未知量 x: <low bits of p+q>
[+] FLAG: SUCTF{congratulation_you_know_small_d_with_hint_factor}
```

## 方法总结
- 核心技巧：Boneh-Durfee / Coppersmith 小根
- 识别信号：小私钥指数与 `p+q` 高位泄露同时存在。
- 复用要点：引入未知低位 `x` 和乘子 `k`，构造二元小根方程；求出 `p+q` 后分解 `N` 解密。
