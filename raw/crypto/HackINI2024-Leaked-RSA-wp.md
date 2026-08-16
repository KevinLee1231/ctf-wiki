# leaked_rsa

## 题目简述

题目给出标准 RSA 参数 `n`、`e`、`c`，另泄露一个由 `p`、`q`、`n` 构造的巨大整数 `eq`。表达式看起来复杂，但大部分项都含有因子 `n`；对 `n` 取模后只剩 `q²`，可以直接恢复素因子。

## 解题过程

源码中的泄露式为：

$$
eq=n^3\bigl(5(p+1)^2+p^4\bigr)+5n(q+p)+q^2
$$

前两项都能被 `n` 整除，因此：

$$
eq\bmod n=q^2\bmod n
$$

生成代码还强制最终满足 $q\le p$。因此 $q^2\le pq=n$；随机素数几乎不可能相等，实际有 $q^2<n$，取模没有改变该值，可以直接计算整数平方根。官方 README 写成三次根，但与源码和 solver 不符，正确操作是平方根。

```python
from Crypto.Util.number import long_to_bytes
from pathlib import Path
from math import isqrt

values = {}
for line in Path("output.txt").read_text().splitlines():
    name, value = line.split("=", 1)
    values[name.strip()] = int(value)

n = values["n"]
e = values["e"]
c = values["c"]
eq = values["eq"]

q_squared = eq % n
q = isqrt(q_squared)
assert q * q == q_squared
assert n % q == 0
p = n // q

phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)
print(long_to_bytes(pow(c, d, n)))
```

输出为：

```text
shellmates{m0dulu5_1s_th3_m0$T_1MPOrT4nt_op3r4tion_1n_cRypt0gr4phY}
```

## 方法总结

- 核心技巧：先对复杂代数泄露式取模，消掉所有含 `n` 的项，再利用参数大小关系做精确整数开方。
- 识别信号：RSA 题给出关于 `p`、`q` 的庞大多项式时，不要先展开；优先尝试对 `n`、`p` 或 `q` 取模化简。
- 复用要点：从模结果开方前必须证明没有发生环绕，并用 `q*q == value` 与 `n % q == 0` 双重验证。
