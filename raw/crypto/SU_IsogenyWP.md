# SU_Isogeny

## 题目简述
题目是 CSIDH/同源映射风格交互。附件 `main.sage` 中的 `cal(A, sk)` 实现私钥向量对应的同源作用，菜单可获取公钥、查询与私钥相关的曲线参数高位，并拿到 `sha256(str(cal(cal(0, pvB), pvA)))` 派生 AES key 后加密的 flag。核心不在密文，而在同源曲线参数之间的代数关系和非法/特殊公钥带来的信息泄露。

## 解题过程
题意

交互提供了一个标准的 CSIDH 风格接口：

- 选项 1 ：给出双方公钥 pkA, pkB

- 选项 2 ：输入两条曲线参数，返回 cal(pkA, pvB) 的高位

- 选项 3 ：给出用真实共享曲线参数导出的 AES-ECB 密文

其中私钥向量只使用奇素数因子，所以整个群作用和 2-isogeny 交换。

关键观察

对 Montgomery 曲线

$$
E_A: y^2 = x^3 + A x^2 + x
$$

它的三个 2-isogenous 邻居里有两条可以写成显式公式：

$$
B = \frac{2(A+6)}{2-A}, \qquad C = \frac{2(A-6)}{A+2} \pmod p
$$

并且三者满足

$$
AB + 2A - 2B + 12 \equiv 0 \pmod p
$$

$$
CB + 2B - 2C + 12 \equiv 0 \pmod p
$$

$$
AC - 2A + 2C + 12 \equiv 0 \pmod p
$$

因为题目里的私钥只走奇素数 isogeny，这些 2-isogeny 关系会和秘密群作用交换。

所以：

- 用 honest pkA 查询 gift，得到真实共享曲线 A = CDH(pkA, pkB) 的高位

- 用 pkA 的两个 2-isogenous 邻居查询 gift，得到对应 B, C 的高位

于是问题变成了论文里的 CI-HNP (Commutative Isogeny Hidden Number Problem)：

已知三条两两 2-isogenous 的共享曲线参数高位，恢复完整共享曲线参数。

为什么能用格

设题目泄露的是高 311 位，低 200 位未知：

$$
A = A_{\text{MSB}} + x,\quad B = B_{\text{MSB}} + y,\quad C = C_{\text{MSB}} + z
$$

其中

$$
0 \le x,y,z < 2^{200}
$$

代回上面的 2-isogeny 关系，得到 3 个模 p 的小根方程：

$$
(A_{\text{MSB}} + x)(B_{\text{MSB}} + y) + 2(A_{\text{MSB}} + x) - 2(B_{\text{MSB}} + y) + 12 \equiv 0 \pmod p
$$

$$
(C_{\text{MSB}} + z)(B_{\text{MSB}} + y) + 2(B_{\text{MSB}} + y) - 2(C_{\text{MSB}} + z) + 12 \equiv 0 \pmod p
$$

$$
(A_{\text{MSB}} + x)(C_{\text{MSB}} + z) - 2(A_{\text{MSB}} + x) + 2(C_{\text{MSB}} + z) + 12 \equiv 0 \pmod p
$$

这正是 ePrint 2023/1409 的 CSIDH 模型。出题人 blog 中也直接把它对应到 CI-HNP：对 CSIDH-512 风格参数，若已知高位比例超过

$$
\frac{13}{24}\approx 54.17\%
$$

就可以用 Automated Coppersmith 在多项式时间内恢复共享曲线。

本题泄露比例是

≈ 60.9%

已经明显超过阈值，所以直接套这个模型即可。

攻击流程

1. 交互拿到 honest pkA, pkB

2. 由 pkA 计算两个 2-isogenous 邻居 pkA_2, pkA_3

3. 分别查询三次 gift，得到 `A >> 200`、`B >> 200`、`C >> 200`。

4. 建立上面的三元小根方程组

5. 用 Automated Coppersmith 恢复 x,y,z

6. 得到完整共享曲线参数 A

7. 取 `key = sha256(str(A).encode()).digest()`，用 AES-ECB 解密 option 3 给出的密文。

实现说明

题解脚本分成两部分：

- `solve_cry3.py`：负责本地/远程交互，构造 2-isogenous 邻居、拿 gift 和密文。
- `recover_cry3.sage`：负责 Automated Coppersmith 恢复共享曲线。

其中 Coppersmith 的辅助实现来自论文作者公开仓库，并做了一个很小的工程化修改：

- 将 Gröbner 提取阶段允许的失败素数次数从 100 提到 3000

这样在本题参数下，m = 6 基本稳定；若失败，再回退到 m = 9 。

参考

- J. Meers, J. Nowakowski, SolvingtheHiddenNumberProblemforCSIDHandCSURFvia

$$
AutomatedCoppersmith, ePrint 2023/1409
$$
- 作者代码仓库：juliannowakowski/automated-coppersmith

cry3_coppersmithsMethod.sage

```python
import os
import shlex
import tempfile
import time
from fpylll import IntegerMatrix

def coppersmithsMethod(polys, modulus, bounds, gbRelations=[], verbose=False,
max_gb_failures=3000):
R = polys[0].parent()

for poly in polys:
if poly.parent() != R:

raise ValueError("Can't instantiate coppersmiths method with
polynomials from different rings.")

tt = cputime()

monList = []
monDict = {}

for poly in polys:
for mon in poly.monomials():
if mon not in monDict:
monDict[mon] = len(monDict)
monList.append(mon)

rows = len(polys)
cols = len(monList)
B = zero_matrix(ZZ, rows, cols)

for i, poly in enumerate(polys):
for mon in poly.monomials():
B[i, monDict[mon]] = int(poly.monomial_coefficient(mon) *
mon(*bounds))

if verbose:
print("Finished basis generation. Polynomials: %d. Time: %fs." %
(len(polys), cputime(tt)), flush=True)

start = time.time()

fd_in, path_in = tempfile.mkstemp(prefix="cry3_basis_", suffix=".tmp")
os.close(fd_in)
fd_out, path_out = tempfile.mkstemp(prefix="cry3_basis_out_",
suffix=".tmp")
os.close(fd_out)
os.unlink(path_out)

try:
with open(path_in, "w+") as handle:
B_str = B.str()
B_str = "\n".join(" ".join(line.split()) for line in
B_str.split("\n"))
handle.write("[\n" + B_str + "\n]")

cmd = "flatter -v %s %s >/dev/null 2>&1" % (shlex.quote(path_in),
shlex.quote(path_out))
success = os.system(cmd)

if success == 0 and os.path.exists(path_out):
B_LLL = matrix(IntegerMatrix.from_file(path_out))
else:
if verbose:
print("flatter not found. Resorting to FPLLL.", flush=True)
B_LLL = B.LLL()
finally:
if os.path.exists(path_in):
os.remove(path_in)
if os.path.exists(path_out):
os.remove(path_out)

stop = time.time()

if verbose:
print("Finished basis reduction. Time: %fs." % (stop - start),
flush=True)

tt = cputime()

solutionPolynomials = list(gbRelations)
for v in B_LLL:
sqNorm = sum(v_i**2 for v_i in v)
norm = RR(sqrt(sqNorm))

if norm < RR(modulus / sqrt(B_LLL.ncols())):
poly = R(0)
for i, mon in enumerate(monList):
poly += R(ZZ(v[i] / mon(*bounds))) * mon
solutionPolynomials.append(poly)

if verbose:
print("Found %d short polynomials. Time: %fs." %
(len(solutionPolynomials), cputime(tt)), flush=True)

tt = cputime()

k = len(R.gens())
if len(solutionPolynomials) < k:
raise RuntimeError("LLL did not find enough short polynomials. Can't
extract solution.")

p = 0
maxBound = max(bounds)
gbModulus = 1
gbFailCounter = 0

crtResults = [[] for _ in range(k)]
moduli = []

while gbModulus < maxBound:
p = next_prime(p + 1)

Rp = R.change_ring(GF(p))
I = Rp * solutionPolynomials

success = True
try:
solutions = I.variety()
except ValueError:
success = False

if success and len(solutions) == 1:
solution = solutions[0]
gbModulus *= p
moduli.append(p)

for i in range(k):
crtResults[i].append(ZZ(solution[Rp.gens()[i]]))
else:
gbFailCounter += 1
if gbFailCounter > max_gb_failures:
raise RuntimeError("Coppersmith heuristic failed. Could not
extract solution from Gröbner basis.")

solutions = [crt(crtResults[i], moduli) for i in range(k)]

if verbose:
print("Finished extracting solutions. Time: %fs." % cputime(tt),
flush=True)

return solutions
```

cry3_optimalShiftPolys.sage

```python
from copy import deepcopy

def getBestShiftPoly(mon, polys, M, poly=1, label=0, best_label=0,
best_poly=1, start=0):
R = polys[0].parent()
n = len(polys)

if label == 0:
label = [0] * n
best_label = [0] * n
best_poly = R(1)

shift_poly = poly * mon

if set(shift_poly.monomials()).issubset(M):
if sum(best_label) <= sum(label):
best_label = label
best_poly = shift_poly

for i in range(start, n):
lm = polys[i].lm()
if mon % lm == 0:
label_new = deepcopy(label)
label_new[i] += 1
poly_new = poly * polys[i]
mon_new = R(mon / lm)
best_label, best_poly = getBestShiftPoly(
mon_new,
polys,
M,
poly_new,
label_new,
best_label,
best_poly,
i,
)

return best_label, best_poly

def constructOptimalShiftPolys(polys, M, modulus, m):
F = []

for mon in M:
label, poly = getBestShiftPoly(mon, polys, M)
poly *= modulus ** (m - sum(label))
F.append(poly)

return F
```

recover_cry3.sage

```python
import sys

load("cry3_coppersmithsMethod.sage")
load("cry3_optimalShiftPolys.sage")

def recover_shared_secret(p, a_msb, b_msb, c_msb, unknown_bits):
R.<x, y, z> = PolynomialRing(QQ, order="lex")

f = (a_msb + x) * (b_msb + y) + 2 * (a_msb + x) - 2 * (b_msb + y) + 12
g = (c_msb + z) * (b_msb + y) + 2 * (b_msb + y) - 2 * (c_msb + z) + 12
h = (a_msb + x) * (c_msb + z) - 2 * (a_msb + x) + 2 * (c_msb + z) + 12

bounds = [2 ** unknown_bits, 2 ** unknown_bits, 2 ** unknown_bits]

last_error = None
for total_m in [6, 9]:
try:
power = total_m // 3
monomials = ((f * g * h) ** power).monomials()
shifts = constructOptimalShiftPolys([f, g, h], monomials, p,
total_m)
low_a, low_b, low_c = coppersmithsMethod(
shifts,
p ** total_m,
bounds,
verbose=True,
max_gb_failures=3000,
)

shared = ZZ(a_msb + low_a)
shared_b = ZZ(b_msb + low_b)
shared_c = ZZ(c_msb + low_c)

if shared >= p or shared_b >= p or shared_c >= p:
raise RuntimeError("Recovered coefficient is not reduced
modulo p.")

if (shared * shared_b + 2 * shared - 2 * shared_b + 12) % p != 0:
raise RuntimeError("Recovered A/B pair does not satisfy the 2-
isogeny relation.")
if (shared_c * shared_b + 2 * shared_b - 2 * shared_c + 12) % p !=
0:
raise RuntimeError("Recovered B/C pair does not satisfy the 2-
isogeny relation.")
if (shared * shared_c - 2 * shared + 2 * shared_c + 12) % p != 0:

raise RuntimeError("Recovered A/C pair does not satisfy the 2-
isogeny relation.")

return int(shared)
except Exception as error:
last_error = error
sys.stderr.write(f"[recover_cry3] total_m={total_m} failed:
{error}\n")
sys.stderr.flush()

raise RuntimeError(last_error)

def main():
if len(sys.argv) != 6:
print("usage: sage recover_cry3.sage <p> <a_msb> <b_msb> <c_msb>
<unknown_bits>", file=sys.stderr)
raise SystemExit(1)

p = ZZ(sys.argv[1])
a_msb = ZZ(sys.argv[2])
b_msb = ZZ(sys.argv[3])
c_msb = ZZ(sys.argv[4])
unknown_bits = int(sys.argv[5])

shared = recover_shared_secret(p, a_msb, b_msb, c_msb, unknown_bits)
print(f"RECOVERED={shared}", flush=True)

main()
```

## 方法总结
- 核心技巧：同源映射参数关系利用
- 识别信号：交互暴露公钥、部分曲线参数或共享值派生密文。
- 复用要点：利用 2-isogeny 与 Montgomery 参数的显式关系构造方程，恢复共享曲线参数后派生 AES key。
