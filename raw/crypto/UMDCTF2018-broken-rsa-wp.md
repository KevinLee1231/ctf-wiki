# UMDCTF 2018 - Broken RSA

## 题目简述

题目给出 PEM 公钥和 128 字节密文。公钥中的指数不是常见的 $65537$，而是 $e=16$；进一步解析后还能发现模数 $n$ 本身是素数，因此这并不是由两个大素数乘积构成的正常 RSA。

## 解题过程

从公钥提取参数后先检查模数：

```python
from Crypto.PublicKey import RSA
from Crypto.Util.number import isPrime

key = RSA.import_key(open("public.key", "rb").read())
print(key.e)          # 16
print(isPrime(key.n)) # True
```

密文满足：

$$
c \equiv m^{16} \pmod n
$$

因为 $n$ 是素数，可以直接在有限域 $\mathbb{F}_n$ 中求 $c$ 的全部 16 次方根。指数 $16$ 与 $n-1$ 不互素，不能像标准 RSA 那样简单计算唯一私钥指数，所以需要枚举全部根并筛选可读明文：

```python
from sage.all import GF, Integer

n = Integer(key.n)
c = Integer(int.from_bytes(open("encrypted_flag.txt", "rb").read(), "big"))

for root in GF(n)(c).nth_root(16, all=True):
    data = int(root).to_bytes((int(root).bit_length() + 7) // 8, "big")
    if b"UMDCTF" in data:
        print(data)
```

可读根为：

```text
UMDCTF-{Wait_you_mean_this_isnt_even_RSA?}
```

## 方法总结

RSA 题不能只看文件格式；必须核对 $n$ 是否为合数、$e$ 是否合理，以及 $\gcd(e,\varphi(n))$ 是否为 $1$。本题的决定性错误是把素数直接当成 RSA 模数，使问题退化为有限域开方。
