# filter plaintext

## 题目简述

服务端公开 80 字节随机秘密的自定义 PCBC 密文，并把 `MD5(secret)` 用作 AES-CBC 密钥加密 flag。解密接口会丢弃任何恰好出现在原秘密中的明文块，试图阻止直接解密秘密。通过在密文前加三个全零块，可以泄露 PCBC 初始向量，并让后续秘密块统一带上一个可消除的异或掩码。

## 解题过程

记 PCBC 初始状态为 $V$，$D(0^{128})=d$。解密分组 $C_i$ 的规则为

$$
P_i=D(C_i)\oplus T_{i-1},\qquad
T_i=C_i\oplus P_i.
$$

向公开的秘密密文前添加三个零分组：

```python
crafted = b"\x00" * 48 + secret_enc
response = decrypt(crafted)
```

前三个分组依次解出：

$$
P'_0=d\oplus V,\qquad P'_1=V,\qquad P'_2=d\oplus V.
$$

因此响应的第二块直接泄露 $V$，第一块与第二块异或即可得到 $d$。当原密文 $C_0,\ldots,C_4$ 接在这三块后面时，状态差会保持为同一个 $d$，故对应输出依次为 $P_i\oplus d$。这些值几乎不可能恰好成为 `secret` 中的原始 16 字节片段，所以不会被过滤掉。

恢复过程为：

```python
pcbc_iv = response[16:32]
mask = xor(response[:16], pcbc_iv)       # D_K(0^128)
secret = xor(response[48:], mask * 5)

key = md5(secret).digest()
cipher = AES.new(key, AES.MODE_CBC, printed_flag_iv)
flag = unpad(cipher.decrypt(flag_enc), 16)
```

这里的 `printed_flag_iv` 是服务端与 flag 密文一起公开的 CBC IV，不要与刚泄露的 PCBC 初始状态混淆。解密得到：

```text
grey{pcbc_d3crypt10n_0r4cl3_3p1c_f41l}
```

## 方法总结

按“是否等于敏感明文”进行输出过滤，无法抵抗具有线性可塑性的链接模式。攻击者可以让所有敏感块统一异或同一掩码，既绕过相等判断，又从已知结构中恢复该掩码。设计解密接口时，应避免返回未经认证的可塑明文；正确做法是使用 AEAD，并在任何明文输出前完成完整性验证。
