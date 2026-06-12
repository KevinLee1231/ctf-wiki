# HD_is_what

## 题目简述

题目是 SIDH/SIKE 风格同源密码题。附件 `task.sage` 生成起始曲线、Alice/Bob 的同源公钥和共享 j-invariant，再用 `sha256(str(shared_j))` 派生 AES-CBC key 加密 flag。题目没有直接输出双方公钥，而是把公钥曲线参数和点坐标拼成长度为 12 的整数向量，再乘上一个由 LCG 生成的整数矩阵进行混淆。

核心同源部分如下：

```python
a, b = 82, 57
p = 2**a * 3**b - 1
Fp2.<i> = GF(p^2, modulus=x^2 + 1)
E_start = EllipticCurve(Fp2, [0, 6, 0, 1, 0])
E_start.set_order((p + 1)^2)

P2, Q2 = get_basis(E_start, 2**a)
P3, Q3 = get_basis(E_start, 3**b)

bobs_key = randint(0, 3**b - 1)
KB = P3 + bobs_key * Q3
phiB = E_start.isogeny(KB, algorithm="factored")
EB = phiB.codomain()
PB, QB = phiB(P2), phiB(Q2)

alices_key = randint(0, 2**a - 1)
KA = P2 + alices_key * Q2
phiA = E_start.isogeny(KA, algorithm="factored")
EA = phiA.codomain()
PA, QA = phiA(P3), phiA(Q3)
```

混淆部分是本题的第一层障碍：

```python
state = int(p)
def next_rand():
    global state
    state = (state * 1664525 + 1013904223) % 2**32
    return state

M_bob = matrix(ZZ, 12, 12)
for r in range(12):
    for c in range(12):
        M_bob[r, c] = (next_rand() % 10 + 10) if r == c else (next_rand() % 5)
Y_bob = vector(ZZ, bob_raw) * M_bob
```

LCG seed 是公开的 `p`，矩阵生成规则也在源码中，因此可以重建矩阵并解线性方程恢复原始公钥。恢复公钥后，再套用 Castryck-Decru attack 攻击 SIDH。

## 解题过程

先从 `output.txt` 读取公开数据：

```python
with open("output.txt", "r") as f:
    data = eval(f.read())

a, b = data["params"]["a"], data["params"]["b"]
p = 2**a * 3**b - 1
```

按附件里的同一套 LCG 规则重建 Bob 的混淆矩阵，并解出原始向量：

```python
state = int(p)

def next_rand():
    global state
    state = (state * 1664525 + 1013904223) % 2**32
    return state

YB = vector(ZZ, data["bob_obfuscated"])
MB = matrix(ZZ, 12, 12)
for r in range(12):
    for c in range(12):
        MB[r, c] = (next_rand() % 10 + 10) if r == c else (next_rand() % 5)

VB = [int(x) for x in MB.solve_left(YB)]
```

注意源码中是：

```python
Y_bob = vector(ZZ, bob_raw) * M_bob
```

所以这里求的是左解 `V * M = Y`。Bob 之后还会继续消耗 LCG 状态生成 Alice 的矩阵，因此恢复 Alice 时不能重置 LCG，要沿用 Bob 矩阵生成后的状态：

```python
YA = vector(ZZ, data["alice_obfuscated"])
MA = matrix(ZZ, 12, 12)
for r in range(12):
    for c in range(12):
        MA[r, c] = (next_rand() % 10 + 10) if r == c else (next_rand() % 5)

VA = [int(x) for x in MA.solve_left(YA)]
```

向量布局来自附件中的序列化函数：

```python
bob_raw  = fp2_to_list(EB.a4()) + fp2_to_list(EB.a6())
bob_raw += point_to_list(PB) + point_to_list(QB)
```

因此恢复出的 `VB` 可解析为 `EB, PB, QB`，`VA` 可解析为 `EA, PA, QA`：

```python
def parse_fp2(lst):
    if len(lst) == 2:
        return Fp2(Fp(lst[0]) + Fp(lst[1]) * i)
    return Fp2(Fp(lst[0]))

def parse_pt(lst, E):
    return E(parse_fp2(lst[:2]), parse_fp2(lst[2:]))

EB = EllipticCurve(Fp2, [0, 6, 0, parse_fp2(VB[0:2]), parse_fp2(VB[2:4])])
PB = parse_pt(VB[4:8], EB)
QB = parse_pt(VB[8:12], EB)

EA = EllipticCurve(Fp2, [0, 6, 0, parse_fp2(VA[0:2]), parse_fp2(VA[2:4])])
PA = parse_pt(VA[4:8], EA)
QA = parse_pt(VA[8:12], EA)
```

接下来使用 [Castryck-Decru-SageMath](https://github.com/GiacomoPope/Castryck-Decru-SageMath) 中的攻击实现。对 Bob 公钥恢复私钥后，在 Alice 曲线上计算共享同源：

```python
load("castryck_decru_attack.sage")

two_i = E_start.isogeny(E_start.lift_x(ZZ(1)), codomain=E_start)
bobs_key = CastryckDecruAttack(E_start, P2, Q2, EB, PB, QB, two_i, num_cores=4)

shared_kernel_B = PA + bobs_key * QA
phi_shared_B = EA.isogeny(shared_kernel_B, algorithm="factored")
shared_j = phi_shared_B.codomain().j_invariant()
```

最后复现服务端的 key 派生并解密：

```python
from hashlib import sha256
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

key = sha256(str(shared_j).encode()).digest()
iv = bytes.fromhex(data["iv"])
ct = bytes.fromhex(data["ciphertext"])
flag = unpad(AES.new(key, AES.MODE_CBC, iv).decrypt(ct), 16)
print(flag)
```

## 方法总结

- 核心技巧：先去掉可逆线性混淆，再对恢复出的 SIDH 公钥使用 Castryck-Decru attack。
- 识别信号：同源题输出的曲线参数和点坐标被线性矩阵混淆，但矩阵由公开 seed 的 LCG 确定。
- 复用要点：遇到“公钥向量乘矩阵”的包装，不要直接进入同源攻击；先检查矩阵是否可重建、是否可逆。恢复出标准 SIDH 公钥格式后，再使用成熟实现攻击私钥并计算共享 j-invariant。
