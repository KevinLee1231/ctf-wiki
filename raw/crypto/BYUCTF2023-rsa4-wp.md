# BYUCTF 2023 - RSA4

## 题目简述

同一明文在三个两两互素的 RSA 模数下使用小指数 `e = 3` 加密，且没有随机填充。这满足 Håstad 广播攻击条件。

## 解题过程

三组密文满足 $c_i\equiv m^3\pmod{n_i}$。

用中国剩余定理恢复 `m^3` 在乘积模数下的唯一值：

```python
N = n1 * n2 * n3
x = 0
for c, n in [(c1, n1), (c2, n2), (c3, n3)]:
    Ni = N // n
    x += c * Ni * pow(Ni, -1, n)
x %= N
```

明文足够短，使真实整数满足 $m^3<n_1n_2n_3$，所以 CRT 结果不是仅有的同余类，而就是整数 $m^3$。对 `x` 取精确整数立方根并转成字节：

```python
from gmpy2 import iroot

m = iroot(x, 3)[0]
plain = int(m).to_bytes((int(m).bit_length() + 7) // 8, 'big')
```

得到：

```text
byuctf{hastad_broadcast_attack_is_why_e_needs_to_be_very_large}
```

## 方法总结

漏洞根因不是“指数 3 必然不安全”，而是小指数、同一明文、多接收者和无随机填充同时成立。现代 RSA 加密必须使用 OAEP 等随机填充，避免确定性广播关系。
