# GreyCTF2023 OT

## 题目简述

服务让用户自选一个 4096 位模数 $N$，随后分别在模 $N$ 与模 $N+23$ 下发布两个 RSA 密文。两条支路加密不同随机秘密 $k_1,k_2$，flag 则由 `SHAKE256(k1 XOR k2)` 派生的流加密。若能让 $N$ 和 $N+23$ 同时易分解，就能恢复两份秘密。

## 解题过程

关键不是分解服务随机生成的 RSA 模数，而是利用“模数由用户提交”这一接口。官方求解器给出了一组可直接复现的构造：

```python
N = (7 * 462770317 * 5694507743) ** 64
small = 2**3 * 3 * 41 * 857 * 1559 * 339023
Q = (N + 23) // small
assert N + 23 == small * Q
```

其中 $Q$ 为素数，且 $N$ 恰为 4096 位；$N$、$N+23$ 都不是素数，能通过服务检查。两边的完整分解为

$N=7^{64}\cdot462770317^{64}\cdot5694507743^{64}$，

$N+23=2^3\cdot3\cdot41\cdot857\cdot1559\cdot339023\cdot Q$。

提交该 $N$ 后，按已知分解计算：

$\varphi(N)=\prod p_i^{e_i-1}(p_i-1)$，

$d=e^{-1}\bmod\varphi(N)$。

两条支路分别执行 RSA 解密：

```python
d1 = inverse(e, phi(factor_N))
d2 = inverse(e, phi(factor_N_plus_23))
k1 = pow(c1, d1, N)
k2 = pow(c2, d2, N + 23)
seed = long_to_bytes(k1 ^ k2)
plain = xor(ciphertext, shake_256(seed).digest(len(ciphertext)))
```

得到：

```text
grey{waitttt_I_thought_factorization_is_hard!!?_bSug9kksE3W9SrPL}
```

## 方法总结

RSA 的“分解困难”只适用于正确生成的模数。协议若允许攻击者指定模数，却只检查位长，就把最关键的安全前提交给了对手。本题还要求同时控制相差 23 的两个数，解题核心因此是预构造可完全分解的数对，而不是对 4096 位随机半素数做通用分解。
