# GreyCTF2022 - Entry

## 题目简述

题目给出一段异或后的密文和官方解题脚本。加密使用长度仅为 4 字节的循环密钥，明文又以已知的 `grey` 开头，因此这是典型的已知明文恢复重复异或密钥问题。

## 解题过程

若密文字节为 $c_i$，明文字节为 $m_i$，循环密钥为 $k_{i\bmod 4}$，则

$$c_i=m_i\oplus k_{i\bmod 4}.$$

利用 flag 固定前缀即可直接恢复完整密钥：

```python
from itertools import cycle

cipher = bytes.fromhex(
    "982e47b0840b47a59c334facab3376a19a1b50ac"
    "861f43bdbc2e5bb98b3375a68d3046e8de7d03b4"
)
known = b"grey"
key = bytes(cipher[i] ^ known[i] for i in range(4))
plain = bytes(c ^ k for c, k in zip(cipher, cycle(key)))
print(key, plain)
```

由于密钥周期恰好等于已知前缀长度，四个密钥字节均能确定。继续对全部密文字节循环异或，得到：

```text
grey{WelcomeToTheGreyCatCryptoWorld!!!!}
```

## 方法总结

重复异或的安全性取决于密钥不能被重复使用且不能过短。看到短周期异或与固定格式明文时，应先用已知前缀求 $k_i=c_i\oplus m_i$，再检查恢复出的密钥是否能在整段密文上产生连贯文本。
