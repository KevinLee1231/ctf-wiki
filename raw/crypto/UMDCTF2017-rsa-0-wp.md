# RSA 0

## 题目简述

题目宣称使用了 4096 bit RSA，但密钥长度并不是安全性的唯一条件。解析 `public_key` 后检查模数 $n$，会发现它恰好是一个完全平方数，即生成者错误地令 $p=q$。

## 解题过程

对于正常 RSA，$n=pq$ 且 $p\ne q$；本题中：

$$
n=p^2
$$

所以直接取整数平方根即可得到因子。由于两个素因子相同，欧拉函数不是常见的 $(p-1)(q-1)$，而应为：

$$
\varphi(p^2)=p(p-1)
$$

随后计算 $d=e^{-1}\bmod\varphi(n)$，对二进制密文按大端转为整数并执行私钥运算。明文使用 PKCS#1 v1.5 填充，可在第二个 `00` 后截取消息：

```python
from math import isqrt
from Crypto.PublicKey import RSA
from Crypto.Util.number import inverse, long_to_bytes

key = RSA.import_key(open("public_key", "rb").read())
n, e = key.n, key.e
p = isqrt(n)
assert p * p == n

phi = p * (p - 1)
d = inverse(e, phi)
c = int.from_bytes(open("encrypted", "rb").read(), "big")
block = long_to_bytes(pow(c, d, n), (n.bit_length() + 7) // 8)

assert block.startswith(b"\x00\x02")
message = block.split(b"\x00", 2)[2]
print(message.decode())
```

输出：

```text
UMDCTF-{NeXt_Tim3_MinD_YouR_Ps_And_qs}
```

该字符串的 SHA-256 与 README 摘要一致。

## 方法总结

超长模数无法弥补密钥生成阶段的结构性错误。检查 RSA 时应先做低成本测试，包括小因子、最大公约数、完全幂和 $p,q$ 是否过近。本题还容易在 $\varphi(n)$ 上犯错：当 $p=q$ 时必须使用 $p(p-1)$。
