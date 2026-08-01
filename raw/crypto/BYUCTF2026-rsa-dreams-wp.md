# RSA Dreams

## 题目简述

题目给出标准 RSA 参数 $n=pq$、公钥指数 $e=65537$ 和密文 $c=m^e\bmod n$，同时额外泄露了 `hint = p + q`。RSA 的安全性依赖于难以从 $n$ 恢复 $p,q$；一旦乘积与和同时已知，两个素数就是一个二次方程的两根，因而无需通用大整数分解。

## 解题过程

由

$$
(x-p)(x-q)=x^2-(p+q)x+pq
$$

可知 $p,q$ 满足

$$
x^2-\text{hint}\cdot x+n=0.
$$

判别式为 $\Delta=\text{hint}^2-4n=(p-q)^2$，因此直接取整数平方根：

```python
import math
from Crypto.Util.number import long_to_bytes

# c、n、e、prime_sum 替换为题目输出
delta = prime_sum * prime_sum - 4 * n
root = math.isqrt(delta)
assert root * root == delta

p = (prime_sum + root) // 2
q = (prime_sum - root) // 2
assert p * q == n

phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)
m = pow(c, d, n)
print(long_to_bytes(m))
```

脚本先用 `p * q == n` 验证根的正确性，再按普通 RSA 流程求私钥指数并解密，得到：

```text
byuctf{great_job_recovering_the_flag}
```

## 方法总结

- 核心技巧：利用已知的 $p+q$ 与 $pq$ 解二次方程恢复 RSA 素因子。
- 识别信号：题目在 $n$ 之外泄露素因子之和、差或其他对称多项式时，应先尝试代数恢复，而不是直接分解 $n$。
- 复用要点：必须用整数平方根并检查判别式为完全平方数，避免浮点二次公式造成大整数精度错误。
