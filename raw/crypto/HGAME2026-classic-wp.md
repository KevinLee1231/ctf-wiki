# Classic

## 题目简述

题目附件给出一个短 RSA 加密脚本和输出数据。脚本生成 512 bit 的 `p, q`，正常计算 `n = p*q`、`c = flag^e mod n`，同时额外泄露 `p` 的高位 `leak = p >> 230`。输出里的 `n`、`leak`、`c` 都是大整数，正文不需要长期全文堆放，关键是这个“泄露 p 高位”的题目定义。

```python
from Crypto.Util.number import *
from secret import flag

p = getPrime(512)
q = getPrime(512)
n = p * q
e = 65537
leak = p >> 230
m = bytes_to_long(flag)
c = pow(m, e, n)
print("n=", n)
print("leak=", leak)
print("c=", c)
```

题目输出解出 RSA 后还会给出第二层提示：明文指向 Vigenere，key 为 `hgame`。

## 解题过程

已知 `p` 的高位后，令 `p = (leak << kbits) + x`，其中 `x` 是未知低位。因为未知部分只有 230 bit，可以把 `x` 作为 `Zmod(n)` 上的一元小根恢复：

```sage
from Crypto.Util.number import *

n = ...      # 输出中的 n
leak = ...   # 输出中的 leak = p >> 230
c = ...      # 输出中的 c
e = 65537
pbits = 512
kbits = pbits - leak.nbits()
p_high = leak << kbits

PR.<x> = PolynomialRing(Zmod(n))
f = x + p_high
roots = f.small_roots(X=2^kbits, beta=0.4, epsilon=0.01)

p = p_high + int(roots[0])
q = n // p
d = inverse(e, (p - 1) * (q - 1))
m = pow(c, d, n)
print(long_to_bytes(int(m)))
```

RSA 解密得到的内容不是最终 flag，而是第二层提示：使用 Vigenere，密钥为 `hgame`。再按该密钥对提示中的密文做 Vigenere 解密即可得到最终结果。这里要记录的是两层结构：第一层通过 `p` 高位泄露恢复 RSA 私钥，第二层按明文提示处理古典密码。

## 方法总结

RSA 素数高位泄露时，把未知低位建成小根问题，比暴力分解更直接。恢复 p 后再看明文是否仍是提示、密文或二次编码；多层 Crypto 题要把每一层输出的语义记录清楚。
