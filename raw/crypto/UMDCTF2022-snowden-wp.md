# Snowden

## 题目简述

服务每次都用新生成的 2048 位 RSA 模数加密同一条消息，但公开指数只在 20 到 31 之间随机选择一个奇数。响应直接给出 $(n,e,c)$，且没有填充。

相同明文、相同小指数和互素模数的多组 RSA 密文满足 Håstad broadcast attack 的条件。

## 解题过程

等待服务返回指数相同的样本。选择最小的可能值 $e=21$，收集 21 组：

$$
c_i\equiv m^{21}\pmod{n_i}.
$$

各次 RSA 密钥独立生成，模数以极高概率两两互素。令 $N=\prod_i n_i$，使用中国剩余定理合并：

$$
M=\sum_i c_iN_i\left(N_i^{-1}\bmod n_i\right)\pmod N,
\qquad N_i=N/n_i.
$$

消息长度足够短，且一共收集了 $e$ 个 2048 位模数，因此 $m^{21}<N$。CRT 结果不是仅与 $m^{21}$ 同余，而就是整数 $m^{21}$ 本身。对 $M$ 求精确 21 次整数根：

```python
from Crypto.Util.number import long_to_bytes
from gmpy2 import iroot

# samples 只保留 e == 21 的 (n, c)
N = 1
for n, _ in samples:
    N *= n

M = 0
for n, c in samples:
    Ni = N // n
    M += c * Ni * pow(Ni, -1, n)
M %= N

m, exact = iroot(M, 21)
assert exact
print(long_to_bytes(int(m)))
```

解出的广播正文末尾为：

```text
UMDCTF{y0u_r3ally_kn0w_y0ur_br04dc45t_4tt4ck!}
```

## 方法总结

- Håstad broadcast attack 不只适用于 $e=3$；只要同一未填充明文在至少 $e$ 个互素模数下使用相同指数，就可 CRT 后开 $e$ 次根。
- 服务会改变指数，不能把不同 $e$ 的样本混在一起；选择最小指数能减少所需连接次数。
- 求根后应检查 `exact`，否则可能是样本数不足、模数不互素、消息过长或解析错误。
- 根因是 textbook RSA 对重复消息没有随机填充；OAEP 会破坏各次加密之间的代数一致性。
