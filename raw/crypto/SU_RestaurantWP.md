# SU_Restaurant

## 题目简述
题目是 tropical semiring 下的矩阵构造题。附件 `main.py` 用 `Point/Block` 抽象实现 min-plus 运算，服务端给出 `cook/eat` 两类交互：`cook(msg)` 会按内部秘密 `fork` 为消息矩阵生成一组矩阵，`eat(msg, A, B, P, R, S)` 则验证选手提交的 JSON 矩阵。目标是提交满足 rank 和取值范围约束的 `A,B,P,R,S`，让 tropical 乘法表达式相等，同时还要满足 `A * B != S`。

题目原型来自 tropical signature 相关构造。预期识别信号是同一秘密 `fork` 会参与多次 `cook` 签名，理论上可以通过约束求解恢复秘密并伪造；本题验证器还存在更直接的构造空间：不恢复 `fork`，而是构造目标 tropical 矩阵 `T`，让右侧各项取最小值后被压成 `T`，同时构造 `A * B = T` 且 `A * B != S`。

## 解题过程
题目信息

题目给了一个 “餐厅” 交互程序，里面有两个重要接口：
- cook(msg) ：正常出菜，返回 A, B, P, R, S
- eat(msg, A, B, P, R, S) ：服务端验菜，如果满足条件就给 flag

交互里真正拿 flag 的逻辑是：

1. 服务端随机生成一个 36 字符串 msg

2. 我们提交 JSON 格式的 A,B,P,R,S

3. 服务端检查：

- `rank(A) >= 7`
- `rank(B) >= 7`
- `rank(P) == rank(R) == rank(S) == 8`
- 所有元素都在 `[0, 256]`
- 在 tropical semiring 下满足：

$$
A * B = (M * fork * M) + (M * P) + (R * M) + S
$$

- 同时要求 `A * B != S`。

这里的 * 和 + 不是普通矩阵乘法和加法，而是 tropical semiring（min-plus 代数）：
- 点加法：a + b = min(a,b)

- 点乘法：`a * b = a + b`
- 矩阵乘法：(A*B)[i][j] = min_k (A[i][k] + B[k][j])

因此这题本质不是常规线性代数，而是 tropical 矩阵构造题。

### 官方/论文视角

出题人给出的预期攻击更接近论文 `A Comprehensive Break of the Tropical Stickel Protocol` 里的思路：向服务端调用两次 `cook`，拿到同一个秘密 `fork` 对两个不同消息矩阵生成的签名，然后把 tropical 方程转成约束求解问题。对每个元素建立取最小值时的约束，使用 Z3 之类的 SMT solver 恢复一组满足条件的 `fork`，最后按正常签名流程伪造 `A,B,P,R,S`。

这条路线的价值在于说明题目本来模拟的是 tropical 签名方案的密钥恢复/伪造问题；但在本题验证器里，`eat` 只检查等式和 rank/range 条件，不强制提交的矩阵必须来自真实 `fork` 签名过程，所以可以走下面这条直接构造路线。

关键观察

### 1.6 服务端真正比较的是两个 tropical 矩阵

设消息矩阵为 M ，则服务端验证的是：

```
W = A * B
Z = (M * fork * M) + (M * P) + (R * M) + S
```

其中右边的 + 也是按元素取最小值，所以：

```
Z[i][j] = min((M*fork*M)[i][j], (M*P)[i][j], (R*M)[i][j], S[i][j])
```

这意味着：

我们不需要知道 fork ，只要能构造别的三项把它压住，让整个最小值固定成我们想要的矩阵即可。

### 1.2 目标矩阵可以取成一个特殊的 rank-1 tropical 形式

定义：

- row_min[i] = min_j M[i][j]
- col_min[j] = min_i M[i][j]

然后选一个向量 y ，满足：
- 0 <= y[j] <= col_min[j]
- row_min[i] + y[j] <= 250
- y 不能全相同

于是定义目标矩阵：

```
T[i][j] = row_min[i] + y[j]
```

因为 fork 的元素非负，所以：

```
(M * fork * M)[i][j] >= row_min[i] + col_min[j] >= row_min[i] + y[j] = T[i][j]
```

也就是说，未知项 M*fork*M 一定不会比 T 更小。

这样我们只要让 (M*P) 、(R*M) 、S 的最小值恰好等于 T ，那么整个 Z 就会被钉死成 T 。

利用思路

目标：构造 A,B,P,R,S ，满足：

```
A * B = T
Z = min(M*fork*M, M*P, R*M, S) = T
```

并同时满足 rank 和元素范围限制。

### 第一步：构造 S

令 `S` 的非对角元等于 `T`，对角元比 `T` 稍微大一点：

```text
S[i][j] = T[i][j]                       (i != j)
S[i][i] = T[i][i] + random(1..20)
```

这样得到的效果：

- 非对角位置由 `S` 直接给出 `T`。
- 对角位置的 `S` 不会成为最小项，需要靠 `M*P` 精确给出 `T[i][i]`。
- 反复随机直到 `rank(S)=8`。

### 第二步：构造 P

目标是让：

```
M * P >= T
且 diag(M * P) = diag(T)
```

做法是：对每一列 j ，找到第 j 行里一个最小值位置 t_j ，令：

```
P[t_j][j] = y[j]
```

其余元素设成更大一些。

这样对于对角元 (j,j) ：

```
(M*P)[j][j] = min_t (M[j][t] + P[t][j])
```

当取到 t=t_j 时：

```
M[j][t_j] + P[t_j][j] = row_min[j] + y[j] = T[j][j]
```

而其他位置都更大，所以能保证：

- 对角元刚好等于 T

- 整体不小于 T

### 第三步：构造 R

目标：

```
R * M >= T
```

让 R[i][t] >= row_min[i] ，于是：

```
(R*M)[i][j] = min_t(R[i][t]+M[t][j]) >= row_min[i] + col_min[j] >= T[i][j]
```

所以 R*M 永远不会比 T 小，只是个托底项。

同时随机到 rank(R)=8 为止。

### 第四步：构造 A,B ，使 A*B=T

脚本把 A 做成 8x7 ，B 做成 7x8 ，并让第 0 个中间维主导：
- A[:,0] = row_min
- B[0,:] = y

这样在 tropical 乘法下，k=0 这一项给出：

```
A[i,0] + B[0,j] = row_min[i] + y[j] = T[i][j]
```

然后对 k=1..6 的所有项，故意让它们都比 T 大：

```
A[i,k] = row_min[i] + random(1..20)
B[k,j] = y[j] + random(0..20)
```

于是：

```
A[i,k] + B[k,j] > row_min[i] + y[j] = T[i][j]
```

最终：

```
(A*B)[i][j] = min_k (A[i,k]+B[k][j]) = T[i][j]
```

同时反复随机直到普通线性代数意义下 rank(A) >= 7 且 rank(B) >= 7 。

为什么一定能过 W == Z and W != S

$$
1. W = A*B = T
$$

由上面的构造直接成立。

$$
2. Z = T
$$

因为：

- M*fork*M >= T

，且对角线等于 T • M*P >= T
- R*M >= T

在非对角线上等于 T ，对角线上大于 T • S

所以对任意位置：

直接把最小值压成 T • 非对角元：S

比 T 大，但 M*P 对角线正好等于 T • 对角元：S

因此整体最小值恰好是 T 。

$$
3. W != S
$$

因为 S 的对角线被故意加大过，而 W=T ，所以 W 不可能等于 S 。

### Exp:

```python
import json
import os
import random
import re
import subprocess
import sys
from hashlib import sha3_512

import numpy as np

def H(x: bytes):
h = sha3_512(x).hexdigest()
return [int(h[i:i+2], 16) for i in range(0, 128, 2)]

def hash_to_M(msg: str) -> np.ndarray:
return np.array(H(msg.encode()), dtype=int).reshape(8, 8)

def trop_mul(X: np.ndarray, Y: np.ndarray) -> np.ndarray:
n, m = X.shape
m2, p = Y.shape
assert m == m2
Z = np.full((n, p), 10**9, dtype=int)
for i in range(n):
for j in range(p):
Z[i, j] = min(int(X[i, k]) + int(Y[k, j]) for k in range(m))
return Z

def build_ab(r: np.ndarray, v: np.ndarray):
W = r[:, None] + v[None, :]

A = np.zeros((8, 7), dtype=int)
A[:, 0] = r
for k in range(1, 7):
A[:, k] = np.minimum(r + 20, 256)
A[k - 1, k] = int(r[k - 1]) + 1

B = np.zeros((7, 8), dtype=int)
B[0, :] = v
for k in range(1, 7):
B[k, :] = np.minimum(v + 20, 256)
B[k, k - 1] = int(v[k - 1]) + 1

assert np.linalg.matrix_rank(A) >= 7
assert np.linalg.matrix_rank(B) >= 7
assert np.array_equal(trop_mul(A, B), W)
return A, B, W

def build_p(M: np.ndarray, W: np.ndarray, v: np.ndarray):
row_argmin = np.argmin(M, axis=1)
for _ in range(10000):
P = np.zeros((8, 8), dtype=int)
for j in range(8):
vals = np.random.randint(int(v[j]) + 1, 257, size=8)
vals[row_argmin[j]] = int(v[j])
P[:, j] = vals
MP = trop_mul(M, P)
if (MP >= W).all() and np.all(np.diag(MP) == np.diag(W)) and
np.linalg.matrix_rank(P) == 8:
return P, MP
raise RuntimeError('build_p failed')

def build_r(M: np.ndarray, W: np.ndarray, r: np.ndarray):
for _ in range(10000):
R = np.zeros((8, 8), dtype=int)
for i in range(8):
vals = np.random.randint(int(r[i]) + 1, 257, size=8)
vals[i] = int(r[i])
R[i, :] = vals
RM = trop_mul(R, M)
if (RM >= W).all() and np.linalg.matrix_rank(R) == 8:
return R, RM
raise RuntimeError('build_r failed')

def build_s(W: np.ndarray, MP: np.ndarray, RM: np.ndarray):
cover = (MP == W) | (RM == W)
slack = 256 - W

cand = np.argwhere(cover & (slack > 0))
if len(cand) == 0:
raise RuntimeError('build_s failed: no cover with slack')

for _ in range(20000):
S = W.copy()
cnt = random.randint(1, min(20, len(cand)))
idxs = np.random.choice(len(cand), size=cnt, replace=False)
for idx in idxs:
i, j = cand[idx]
S[i, j] += random.randint(1, int(slack[i, j]))
if np.linalg.matrix_rank(S) == 8 and not np.array_equal(S, W):
return S
raise RuntimeError('build_s failed')

def forge_payload(msg: str):
M = hash_to_M(msg)
r = M.min(axis=1)
c = M.min(axis=0)

# 关键：取 v_j <= col_min_j，并强制 r_i + v_j <= 256，
# 这样 W 本身就能直接作为合法的 S 基底提交。
vmax = 256 - int(r.max())
v = np.minimum(c, vmax)

A, B, W = build_ab(r, v)
P, MP = build_p(M, W, v)
R, RM = build_r(M, W, r)
S = build_s(W, MP, RM)

return {
'A': A.tolist(),
'B': B.tolist(),
'P': P.tolist(),
'R': R.tolist(),
'S': S.tolist(),
}

def run_local_once():
proc = subprocess.Popen(
[sys.executable, '-u', 'main.py'],
cwd=os.path.dirname(__file__),
stdin=subprocess.PIPE,
stdout=subprocess.PIPE,
stderr=subprocess.STDOUT,
text=True,
bufsize=1,

)

def read_until(token: str):
buf = ''
while token not in buf:
ch = proc.stdout.read(1)
if not ch:
break
buf += ch
return buf

sys.stdout.write(read_until('>>> '))
proc.stdin.write('2\n')
proc.stdin.flush()

buf = read_until('>>> ')
sys.stdout.write(buf)
m = re.search(r'Please make (.+?) for me!', buf)
if not m:
raise RuntimeError('challenge not found')
msg = m.group(1)
print(f'[solver] challenge = {msg}')

proc.stdin.write(json.dumps(forge_payload(msg)) + '\n')
proc.stdin.flush()

out = ''
while True:
ch = proc.stdout.read(1)
if not ch:
break
out += ch
if 'FLAG:' in out or 'This is not what I wanted!' in out or 'These are
illegal food ingredients' in out:
# 再读到本行结束
while True:
ch2 = proc.stdout.read(1)
if not ch2:
break
out += ch2
if ch2 == '\n':
break
break
sys.stdout.write(out)

if __name__ == '__main__':
run_local_once()
```

## 方法总结
- 核心技巧：tropical 矩阵构造；预期方向还可以抽象成 tropical 签名的约束求解伪造。
- 识别信号：矩阵运算不是普通代数，而是 min-plus；验证条件同时包含 rank、范围和不等式，且只检查最终矩阵关系。
- 复用要点：若验证器只检查等式，可先构造目标 tropical 矩阵，再分别构造 `S/P/R/A/B` 满足等式和 rank 条件；若必须遵循真实签名流程，则应利用多组 `cook` 输出建立 tropical min 约束，用 SMT 求解隐藏矩阵后再伪造。
