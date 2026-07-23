# Low Effort Required

## 题目简述

题目给出 RSA 公钥参数与密文，公开指数为 $e=5$。明文过短且没有填充，使得 $m^5<n$；模运算实际上没有发生，密文就是普通整数 $m^5$。

## 解题过程

RSA 加密为：

$$
c \equiv m^e \pmod n
$$

当 $m^5<n$ 时，有 $c=m^5$。因此直接求精确五次整数根，再把整数转回大端字节：

```python
import gmpy2
from pathlib import Path
from Crypto.Util.number import long_to_bytes

public_key = Path("public_key").read_text().strip()
n_text, e_text = public_key.split(":")
n, e = int(n_text), int(e_text)
c = int(Path("ciphertext").read_text())

m, exact = gmpy2.iroot(c, e)
assert exact
assert pow(int(m), e) == c
print(long_to_bytes(int(m)).rstrip(b"\x00").decode())
```

得到：

```text
UMDCTF-{f1x_y0ur_3xp0s}
```

## 方法总结

低指数 RSA 必须配合安全填充。看到较小的 $e$ 时，应先比较密文、模数和可能的明文长度；若 $m^e<n$，问题会退化为普通整数开根，无须分解 $n$。
