# Blindfolded

## 题目简述

服务使用未填充的 textbook RSA，并同时提供加密与解密接口。它只拒绝对“原始 flag 密文”进行解密，却允许解密经过乘法变形的密文。RSA 的乘法同态使这种黑名单无效；此外，加密接口还能在不知道 $N$ 的情况下恢复模数。

## 解题过程

公开指数为 $e$，加密接口返回 $E(m)=m^e\bmod N$。因此每个查询都满足：

$$
N\mid m^e-E(m).
$$

对多个互素明文查询后取差值的最大公因数，通常即可消掉偶然倍数并得到 $N$：

~~~python
from math import gcd

values = []
for m in (2, 3, 5, 7):
    values.append(pow(m, e) - encrypt(m))

N = values[0]
for value in values[1:]:
    N = gcd(N, value)
~~~

设 flag 明文为 $m$、给出的密文为 $c=m^e\bmod N$。任选非零系数 $k=2$，构造：

$$
c' = c\cdot k^e\bmod N = (mk)^e\bmod N.
$$

$c'$ 与被禁的 $c$ 不同，服务会返回 $mk\bmod N$。再请求解密 $k^e\bmod N$ 得到 $k$，或直接计算 $k^{-1}\bmod N$，即可恢复：

~~~python
modified = c * pow(2, e, N) % N
twice_message = decrypt(modified)
message = twice_message * pow(2, -1, N) % N
print(long_to_bytes(message))
~~~

本题明文足够小，不会在乘 2 后发生影响结果的模回绕，最终得到：

~~~text
maple{d0nt_f0rg37_t0_h@sh!!}
~~~

## 方法总结

textbook RSA 是确定性的，而且具有可塑性，不能靠“拒绝某个密文”阻止选择密文攻击。实际系统应使用经过证明的填充与封装方案，例如 RSA-OAEP，并且不应向攻击者同时暴露任意解密 oracle。看到 RSA 加密 oracle 时，也应想到利用 $m^e-E(m)$ 的公因数恢复隐藏模数。
