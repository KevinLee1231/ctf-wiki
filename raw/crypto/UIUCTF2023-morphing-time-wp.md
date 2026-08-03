# UIUCTF 2023 Morphing Time Writeup

## 题目简述

服务实现了乘法形式的 ElGamal。公开参数为素数 $p$、生成元候选 $g=2$ 和公钥 $A=g^a\bmod p$。对消息整数 $m$，随机选择 $k$ 后生成：

$$
\operatorname{Enc}(m)=(g^k,\;mA^k)=(c_1,c_2)\pmod p.
$$

服务先给出 flag 的密文 $(c_1,c_2)$，再允许选手提交另一个密文 $(c'_1,c'_2)$。它不会直接解密选手提交的密文，而是返回分量乘积 $(c_1c'_1,c_2c'_2)$ 的解密结果。

## 解题过程

ElGamal 在此消息表示下具有乘法同态性。若：

$$
\operatorname{Enc}(m_1)=(g^k,m_1A^k),
$$

$$
\operatorname{Enc}(m_2)=(g^{k'},m_2A^{k'}),
$$

则分量相乘得到：

$$
(g^{k+k'},m_1m_2A^{k+k'})=\operatorname{Enc}(m_1m_2).
$$

因此，可以自行选择一个已知且在模 $p$ 下可逆的消息 $m_0$，用公开参数正常加密它，再把所得密文提交给 oracle。服务返回的不是 flag 本身，而是：

$$
r\equiv flag\cdot m_0\pmod p.
$$

因为 flag 整数和所选 $m_0$ 都远小于 $p$，且 $m_0\ne0\pmod p$，计算 $r\cdot m_0^{-1}\bmod p$ 即可恢复 flag。核心脚本如下：

```python
from random import randint
from Crypto.Util.number import long_to_bytes
from pwn import remote

io = remote("morphing.chal.uiuc.tf", 1337)

io.recvuntil(b"g = ")
g = int(io.recvline())
io.recvuntil(b"p = ")
p = int(io.recvline())
io.recvuntil(b"A = ")
A = int(io.recvline())

io.recvuntil(b"c1 = ")
flag_c1 = int(io.recvline())
io.recvuntil(b"c2 = ")
flag_c2 = int(io.recvline())

known = int.from_bytes(b"knownplaintext", "big")
k = randint(2, p - 1)
chosen_c1 = pow(g, k, p)
chosen_c2 = known * pow(A, k, p) % p

io.sendlineafter(b"c1_ = ", str(chosen_c1).encode())
io.sendlineafter(b"c2_ = ", str(chosen_c2).encode())
io.recvuntil(b"m = ")
product_plaintext = int(io.recvline())

flag_int = product_plaintext * pow(known, -1, p) % p
print(long_to_bytes(flag_int).decode())
```

其中 `flag_c1` 和 `flag_c2` 只用于同步并记录服务返回值；真正的密文乘法由服务端完成。运行后得到：

```text
uiuctf{h0m0m0rpi5sms_ar3_v3ry_fun!!11!!11!!}
```

## 方法总结

本题利用的是加密方案的可塑性，而不是求解离散对数。服务虽然避免了直接解密目标密文，却允许攻击者控制一个与目标密文相乘的因子，并泄露乘积明文，等价于提供了足以恢复目标的选择密文 oracle。防御时应使用具备完整性和不可塑性的认证加密设计，并禁止对任何由目标密文可计算得到的相关密文返回解密结果。
