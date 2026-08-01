# BYUCTF 2023 - RSA5

## 题目简述

同一明文在同一个模数 $n$ 下分别使用 $e_1=65537$、$e_2=65521$ 加密。两个指数互素，因此可做 RSA common modulus attack。

## 解题过程

扩展欧几里得算法求整数 $a,b$，使 $ae_1+be_2=1$。于是
$c_1^a c_2^b\equiv m^{ae_1+be_2}\equiv m\pmod n$。

实现时负指数要转为模逆：

```python
def modpow_signed(c, k, n):
    return pow(c, k, n) if k >= 0 else pow(pow(c, -1, n), -k, n)

m = modpow_signed(c1, a, n) * modpow_signed(c2, b, n) % n
```

转换为字节后得到：

```text
byuctf{NEVER_USE_SAME_MODULUS_WITH_DIFFERENT_e_VALUES}
```

## 方法总结

不同公钥指数不能补救模数复用；当同一消息在同一 `n` 下加密且指数互素时，Bézout 系数会直接消去指数。每个 RSA 密钥对都应拥有独立生成的素因子与模数，并使用随机填充。
