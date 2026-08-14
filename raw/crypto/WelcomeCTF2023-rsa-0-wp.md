# RSA-0

## 题目简述

题目给出标准 RSA 参数 $N,e,c$，同时直接泄露了一个素因子 $p$。RSA 的安全性依赖于无法从 $N=pq$ 分解出 $p,q$；一旦任意一个素因子泄露，就能恢复私钥。

## 解题过程

由 $q=N/p$ 得到另一个素因子，再计算：

$$
\varphi(N)=(p-1)(q-1),\qquad d=e^{-1}\bmod\varphi(N).
$$

最后计算 $m=c^d\bmod N$ 并转回字节串：

```python
from Crypto.Util.number import long_to_bytes

# c、N、p 替换为题目输出
e = 0x10001
q = N // p
phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)
m = pow(c, d, N)
print(long_to_bytes(m))
```

输出为：

```text
greyhats{HelloWorld_from_RSA_jHZpVMPf8CnwWf8s}
```

## 方法总结

- 核心技巧：已知 RSA 一个素因子时直接重建 $\varphi(N)$ 和私钥指数。
- 识别信号：题目同时给出 $N$ 与其因子，或泄露的信息足以计算出因子。
- 复用要点：恢复明文时注意大整数与字节串的转换，并确认 $e$ 在 $\varphi(N)$ 下可逆。
