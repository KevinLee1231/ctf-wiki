# Classic_CrackMe

## 题目简述

这是一个 .NET CrackMe。合法输入共 46 个字符，格式为 `hgame{<24 字符 Base64 IV><15 字符后缀>}`。程序用同一 AES 密钥做两次 CBC 校验：第一项约束 IV，第二项约束后缀。CBC 首块关系可以把两个约束分别逆解。

## 解题过程

从反编译代码取出常量：

```text
key              = SGc0bTNfMm8yMF9XZWVLMg==
original IV      = MFB1T2g5SWxYMDU0SWN0cw==
original cipher  = mjdRqH4d1O8nbUYJk+wVu3AeE7ZtE9rtT/8BA8J897I=
known plaintext  = Same_ciphertext_
target block     = dJntSWSPWbWocAq4yjBP5Q==
```

CBC 第一块满足 $C_1=E_K(P_1\oplus IV)$。若已知原明文块 $P$、原 IV 和希望解出的明文块 $P'$，使同一个 $C_1$ 在新 IV 下解出 $P'$，则：

$$
IV'=IV\oplus P\oplus P'.
$$

完整计算如下：

```python
from base64 import b64decode, b64encode
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad, unpad

key = b64decode("SGc0bTNfMm8yMF9XZWVLMg==")
iv = b64decode("MFB1T2g5SWxYMDU0SWN0cw==")
old_ct = b64decode("mjdRqH4d1O8nbUYJk+wVu3AeE7ZtE9rtT/8BA8J897I=")
target_c2 = b64decode("dJntSWSPWbWocAq4yjBP5Q==")

old_plain = unpad(AES.new(key, AES.MODE_CBC, iv).decrypt(old_ct), 16)
known = b"Same_ciphertext_"
new_iv = bytes(a ^ b ^ c for a, b, c in zip(old_plain, iv, known))

c1 = AES.new(key, AES.MODE_CBC, iv).encrypt(pad(known, 16))[:16]
plain = unpad(AES.new(key, AES.MODE_CBC, iv).decrypt(c1 + target_c2), 16)
suffix = plain[16:]

flag = b"hgame{" + b64encode(new_iv) + suffix + b"}"
print(flag.decode())
```

程序先解出原明文 `Learn principles`，随后得到：

```text
new IV (Base64): L1R5WFl6UG5ZOyQpXHdlXw==
suffix:           DiFfer3Nt_w0r1d
```

最终输入为：

```text
hgame{L1R5WFl6UG5ZOyQpXHdlXw==DiFfer3Nt_w0r1d}
```

## 方法总结

- 核心技巧：利用 CBC 第一块的异或关系控制新 IV，再把已知第一密文块与目标第二块拼接解出后缀。
- 识别信号：同一密钥、可控 IV、已知明文和固定密文同时出现时，优先写出 CBC 的逐块方程。
- 复用要点：Base64 填充符也是输入的一部分；应先验证解码后长度为 16 或 32 字节，再进行 AES 运算。
