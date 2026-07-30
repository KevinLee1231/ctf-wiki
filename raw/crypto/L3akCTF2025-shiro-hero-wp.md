# L3akCTF 2025 Shiro Hero Writeup

## 题目简述

题目使用一个名为 `xorshiro256` 的 256 位 PRNG 生成 ECDSA nonce。程序先泄露四次 `next_raw()` 的结果，再用下一次带 temper 的输出作为 secp256k1 签名 nonce $k$：

```python
state = [randbits(64) for _ in range(4)]
prng = xorshiro256(state)
leaks = [prng.next_raw() for _ in range(4)]
sig = ECDSA.ecdsa_sign(H, Apriv, prng)
```

私钥 $d$ 经 `SHA256(long_to_bytes(d))` 派生 AES-CBC 密钥。只要从四个 64 位泄露恢复 PRNG 状态并预测 $k$，就能由一条 ECDSA 签名反推出私钥。

## 解题过程

### 确认泄露与签名输出的对齐

状态更新为：

```python
s0, s1, s2, s3 = state
t = (s1 << 17) & MASK64

s2 ^= s0
s3 ^= s1
s1 ^= s2
s0 ^= s3
s2 ^= t
s3 = rotl(s3, 45)
state = [s0, s1, s2, s3]
```

`next_raw()` 返回更新后的 `s1`。签名时调用 `prng()`，即先执行下一轮 `next_raw()`，再计算

$$
\operatorname{temper}(x)
=\operatorname{rotl}_{64}(5x,7)\cdot9\bmod2^{64}.
$$

四个泄露合计正好 256 位，与完整状态大小相同。虽然各状态字不是直接按顺序泄露，但更新只有异或、移位和旋转，可以用位向量约束精确恢复。

### 用 Z3 恢复状态

建立四个 64 位未知量，符号执行状态转移。每一轮都约束更新后的 `s1` 等于对应泄露：

```python
s0, s1, s2, s3 = BitVecs("s0 s1 s2 s3", 64)
state = [s0, s1, s2, s3]

for leak in leaks:
    state = symbolic_transition(state)
    solver.add(state[1] == leak)

assert solver.check() == sat
seed = [solver.model().eval(x).as_long()
        for x in (s0, s1, s2, s3)]
```

官方预测器采用“先返回当前 `s1`，再更新状态”的常见实现，而题目采用“先更新，再返回新 `s1`”。它把第一次泄露视为等价起始状态的当前输出，并在消费四个泄露后以旧状态字做 temper；两种写法在要预测的那个值上对齐。自己重写 solver 时必须明确这一点，否则很容易多推进或少推进一轮。

预测得到：

```text
k = 11724664478660438594
```

### 由 nonce 恢复 ECDSA 私钥

题目虽然把变量命名为 `Hash`，实际传给签名函数的是

```python
H = bytes_to_long(message)
```

而不是消息摘要。恢复时必须使用同一个大整数。ECDSA 签名关系为

$$
s=k^{-1}(H+rd)\pmod n,
$$

所以

$$
d=(sk-H)r^{-1}\pmod n.
$$

```python
d = ((s * k - H) * inverse(r, ECDSA.n)) % ECDSA.n
```

恢复出的私钥为：

```text
0xde63dc64b1de88459f14e4b05a1f6f22fbb94be7088a01f4af13b21ce31b7b70
```

可以再验证 $dG$ 是否等于题目给出的公钥，排除状态对齐错误。

### 解密

```python
key = hashlib.sha256(long_to_bytes(d)).digest()
raw = bytes.fromhex(ciphertext)
iv, ct = raw[:16], raw[16:]
flag = unpad(AES.new(key, AES.MODE_CBC, iv).decrypt(ct), 16)
```

在仓库官方环境中运行 solver，签名验证、nonce 预测、私钥恢复和解密均成功，输出：

```text
L3AK{u_4r3_th3_sh1r0_h3r0!}
```

## 方法总结

ECDSA 的 nonce 必须不可预测。即使私钥本身来自独立随机源，只要 $k$ 的 PRNG 状态被完整泄露，一条签名就足以恢复私钥。本题还需要特别处理实现差异：泄露函数返回更新后的状态字，官方预测器则按返回旧状态字的约定建模，二者必须在签名所用 nonce 处正确对齐。
