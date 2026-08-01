# Times

## 题目简述

题目在自定义椭圆曲线上计算公开点 $P$ 的标量乘法 $Q=1337P$，用 $Q$ 的 $x$ 坐标派生 AES 密钥，再以 AES-CBC 加密 flag。曲线、点、标量、IV 与密文均已公开，所以这里没有离散对数问题，只需正确复现曲线运算。

## 解题过程

从 `ellipticcurve.py` 可确定有限域模数、曲线参数以及点加法规则。按二进制 double-and-add 计算：

```python
def mul(k, p):
    out = None
    cur = p
    while k:
        if k & 1:
            out = add(out, cur)
        cur = add(cur, cur)
        k >>= 1
    return out

q = mul(1337, base_point)
```

生成端把 `str(q.x)` 做 SHA-1，并取十六进制摘要前 16 个字符作为 16 字节 AES 密钥。解密必须完全复现这个字符串化过程，而不是直接使用整数的二进制表示：

```python
from hashlib import sha1
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

key = sha1(str(q.x).encode()).hexdigest()[:16].encode()
plain = unpad(AES.new(key, AES.MODE_CBC, iv).decrypt(ciphertext), 16)
print(plain.decode())
```

得到：

```text
byuctf{mult1pl1c4t10n_just_g0t_s0_much_m0r3_c0mpl1c4t3d}
```

## 方法总结

看到椭圆曲线并不等于需要求离散对数。本题标量已知，决定性工作是忠实复现点乘、整数到字符串的编码以及密钥截取方式。密码题中任何一次表示转换都属于算法的一部分。
