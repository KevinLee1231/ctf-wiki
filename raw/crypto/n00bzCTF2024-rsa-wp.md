# RSA

## 题目简述

RSA 使用 $e=3$ 直接加密 flag，没有填充。消息足够短，使 $m^3<n$，所以模约简根本没有发生，密文就是普通整数立方。

## 解题过程

对密文取精确整数立方根，并检查结果确实为整数根：

```python
from Crypto.Util.number import long_to_bytes
from gmpy2 import iroot

m, exact = iroot(c, 3)
assert exact
print(long_to_bytes(int(m)).decode())
```

得到：

```text
n00bz{crypt0_1s_1nc0mpl3t3_w1th0ut_rs4!!}
```

## 方法总结

这不是分解模数，而是 textbook RSA 的低指数小消息攻击。实际加密必须使用 RSA-OAEP 等随机填充，使编码后的消息不再满足可直接开根的条件。
