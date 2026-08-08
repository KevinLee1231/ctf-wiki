# MiniLCTF2022 R1ngWin Writeup

## 题目简述

题目使用 `py-fhe` 的 BFV 实现在商环

$$
R_q=\mathbb Z_q[x]/(x^{32}+1)
$$

中加密 32 字节 flag，参数为明文模数 $257$、密文模数 $q=\mathtt{0x9000000000000}$。私钥多项式 $s$ 的系数取自 $\{-1,0,1\}$，而出题修改把公钥误差固定为 3 的倍数。由于 $3\mid q$，可通过模 3 消去误差并直接恢复私钥。

## 解题过程

修改后的密钥生成代码满足

$$
p_0=-(p_1s+3e)\pmod q.
$$

因为 $q$ 本身是 3 的倍数，把所有系数映射到

$$
R_3=\mathbb F_3[x]/(x^{32}+1)
$$

后，误差项消失：

$$
p_0\equiv-p_1s\pmod3.
$$

附件中的 $p_1$ 在该商环可逆，因此

$$
s=-p_0p_1^{-1}.
$$

把恢复出的系数 $2$ 映射回 $-1$，得到完整三值私钥：

```text
[0, 1, 0, 1, 0, -1, 0, 1, 1, 1, 0, -1, 0, 0, 0, 0,
 0, 1, 1, 1, 0, 0, 1, 0, -1, 1, 1, 0, -1, 1, -1, 1]
```

随后按原库构造 `SecretKey` 与 `Ciphertext`，执行 BFV 解密和批编码解码：

```sage
F = GF(3)
R.<x> = PolynomialRing(F)
Q.<a> = R.quotient(x^32 + 1)

s_mod3 = -Q(p0) / Q(p1)
s = [(-1 if int(s_mod3[i]) == 2 else int(s_mod3[i])) for i in range(32)]

params = BFVParameters(32, 257, 0x9000000000000)
sk = SecretKey(Polynomial(32, s))
ct = Ciphertext(Polynomial(32, c0), Polynomial(32, c1))
plain = BFVDecryptor(params, sk).decrypt(ct)
flag = bytes(BatchEncoder(params).decode(plain))
```

本地还验证了 $p_0+p_1s\equiv0$ 于 $R_3$，解密结果为：

```text
miniLCTF{s3@rch1ng_R1w3_s0_c001}
```

## 方法总结

RLWE/BFV 的安全性依赖“小误差存在但不能被简单消去”。若密文模数与误差的公共因子可知，将公钥降模便可能把带噪声方程变成精确线性方程。本题恢复私钥后仍应回到原参数和原批编码器解密，不能把 BFV 明文多项式系数直接当字节；批编码包含一次有限域变换。
