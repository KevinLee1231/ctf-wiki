# Are S A?

## 题目简述

题目给出 RSA 公钥参数 $n,e$ 与密文 $c$。异常之处在于 `keygen.py` 只生成了一个大素数，并直接令它充当模数 $n$，而不是通常的 $n=pq$。因此分解模数不是本题的障碍；只要确认 $n$ 为素数，就能直接写出欧拉函数。

## 解题过程

对素数 $n$，有

$$
\varphi(n)=n-1.
$$

RSA 私钥指数仍满足 $ed\equiv1\pmod{\varphi(n)}$，所以直接计算 $d=e^{-1}\bmod(n-1)$，再做模幂即可：

```python
from Crypto.Util.number import long_to_bytes

n, e, c = ...  # 从 cne.txt 读取
d = pow(e, -1, n - 1)
m = pow(c, d, n)
print(long_to_bytes(m))
```

解出的明文为：

```text
byuctf{d1d_s0m3_rs4_stuff...m1ght_d3l3t3_l4t3r}
```

## 方法总结

RSA 的核心前提是能正确求出 $\varphi(n)$，不必机械地把所有模数都当成两个素数的乘积。先检查密钥生成代码与模数性质；当 $n$ 本身为素数时，$\varphi(n)=n-1$，私钥可直接恢复。
