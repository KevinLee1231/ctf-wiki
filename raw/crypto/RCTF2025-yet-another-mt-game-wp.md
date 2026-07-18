# yet another MT game

## 题目简述

服务生成 64 字节随机 `secret`，把它解释成整数后调用 SageMath 的 `set_random_seed()`，再允许选手执行一次：

```python
random_matrix(Zmod(mod), nrow, ncol).list()
```

输出总信息量不得超过 19937 bit。最后若提交的十六进制字节串等于 `secret` 即可获得 flag。决定性问题是：Sage 并不只有 Python 的 MT19937；小模数矩阵直接消耗 GMP 的 MT19937，而 `set_random_seed()` 初始化的正是这套 GMP 状态。

## 解题过程

### 确认随机矩阵使用哪一套 PRNG

源码调用关系如下：

```text
random_matrix(..., algorithm='randomize')
  -> 具体矩阵类型的 randomize()
```

模数足够小时，Sage 选择 dense mod-n 矩阵，其模板实现逐项执行：

```text
current_randstate().c_random() % mod
```

`c_random()` 内部调用 `gmp_urandomb_ui(gmp_state, 31)`，所以这里泄露 GMP randstate 的输出。模数较大而退化为通用矩阵时，元素来自 `Zmod(mod).random_element()`，最终使用 `sage.misc.prandom`，也就是 Python `random.Random`。选择 `mod = 2` 或 `mod = 2^16` 可以稳定留在前一种路径。

这一区分很重要：Sage 的 Python MT 是第一次使用时才创建，其 128-bit 种子也取自 GMP randstate；即使恢复了它，也不能直接反推出由 `os.urandom(64)` 产生的 512-bit `secret`。本题必须恢复 GMP MT 的初始化状态。

### 将 oracle 输出写成 GF(2) 方程

MT19937 的状态由 624 个 32-bit word 构成，twist 和输出均为 GF(2) 上的线性运算。若令 19968 个状态位均为符号变量，就可以符号执行相同的 twist/temper 过程，并把 oracle 泄露的低位写成线性方程。

有两种等价取样方式：

- PDF 的 EXP 发送 `mod=2, nrow=1, ncol=19937`，得到连续 19937 个最低位；
- 官方单题 WP 建议发送 `mod=2^16` 并取 1246 个元素，得到 `16 * 1246 = 19936` 个低位方程，再枚举剩余 1 bit。

GMP 初始化后的 `mt[0]` 只有最高位可能为 1，低 31 位固定为 0；这些已知零位也应作为约束。还必须复现 GMP 的 2000 次 warm-up，使符号状态与服务第一项输出严格对齐。利用 `gf2bv` 一类 bit-vector 线性求解器时，核心结构是：

```python
lin = LinearSystem([32] * 624)
mt = lin.gens()
rng = MT19937(mt)

equations = [mt[0] & 0x7fffffff]  # 31 个固定零位
align_gmp_warmup(rng, 2000)

for leaked_value in output:
    equations.extend(symbolic_low_bits(rng, bits_per_value) ^ leaked_value)

for state in lin.solve_all(equations):
    # 将 mt[1]..mt[623] 和 mt[0] 的最高位拼回 seed2
    ...
```

`gf2bv` 不是密码攻击本身，只是把 MT 的线性位运算自动展开并解 GF(2) 方程；其源码与 MT 模型可从 [maple3142/gf2bv](https://github.com/maple3142/gf2bv) 获取。即使换成自写高斯消元，所解的方程完全相同。

### 从 GMP 状态反推原始 seed

GMP 的 `rand/randmts.c` 并非像 CPython 那样直接扩展 seed，而是先执行：

```python
P = 2**19937 - 20023
E = 1074888996

seed1 = seed % (2**19937 - 20027) + 2
seed2 = pow(seed1, E, P)
```

`seed2` 被从低位开始拆成 `mt[1] ... mt[623]`，最后一个剩余 bit 放到 `mt[0]` 的最高位；随后状态经过 warm-up。因此线性求解所得的初始化 word 可以直接拼回这个 19937-bit `seed2`。

反推指数时有：

```text
gcd(E, P - 1) = 12
```

不能直接求 `E` 在 `P-1` 下的逆。令：

```python
I = pow(E // 12, -1, (P - 1) // 12)
seed1_pow_12 = pow(seed2, I, P)
seed1, exact = gmpy2.iroot(seed1_pow_12, 12)
```

这里整数开 12 次根之所以成立，是因为真实 `seed` 只有 512 bit。它远小于两个 19937-bit 模数，所以 `seed1 = seed + 2`，且 `seed1^12` 仍小于 `P`；模幂结果没有发生模回绕。对满足状态固定 bit、且 `iroot(..., 12)` 返回精确根的候选，执行 `secret_int = seed1 - 2`，再编码成固定 64 字节：

```python
secret = int(secret_int).to_bytes(64, 'big')
io.sendlineafter(b'secret', secret.hex().encode())
```

服务逐字节比较该值与原始 `os.urandom(64)`；正确候选会返回 flag。

## 方法总结

本题需要同时审计三层实现：Sage 根据矩阵类型选择的随机源、GMP MT19937 的状态输出、以及 GMP 特有的 seed 映射。只看到“MT19937”就套 CPython 的 624 个完整输出恢复会走错方向。稳定路线是先用安全的小模数确保泄露 GMP 输出，再用 GF(2) 方程恢复精确对齐的状态，最后利用原始 seed 只有 512 bit 这一上界，把不可逆的指数映射化成“部分指数逆 + 整数 12 次根”。
