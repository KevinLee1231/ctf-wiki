# 算法祭司

## 题目简述

附件是一个 .NET 程序。程序从资源中读取 8 字节字符串，派生 DES 密钥后加密用户输入，并将 Base64 结果与内置密文比较；需要准确区分密钥与 IV 的来源并反向解密。

## 解题过程

在 ILSpy 中查看 `Main`，可以还原出以下关键数据流：

- 资源 `encryptedKey` 的值为 `STV>!'+#`；
- 每个字符与 `0x66` 异或，得到 8 字节 DES 密钥 `520XGAME`；
- `Key` 使用异或后的 `520XGAME`，但 `IV` 使用原始的 `STV>!'+#`；
- `DESCryptoServiceProvider` 默认采用 CBC 模式和 PKCS#7 填充；
- 待比较的 Base64 密文为 `s7/e+JnJbGEdE9j2g3XHxgym+G6Fu/PjJuW80NeMKgemdqaWG9KVM8Tfcc0eRfaA`。

按程序参数复现解密：

```python
from base64 import b64decode

from Crypto.Cipher import DES
from Crypto.Util.Padding import unpad

encrypted_key = "STV>!'+#"
key = bytes(ord(ch) ^ 0x66 for ch in encrypted_key)
iv = encrypted_key.encode()
ciphertext = b64decode(
    "s7/e+JnJbGEdE9j2g3XHxgym+G6Fu/PjJuW80NeMKgemdqaWG9KVM8Tfcc0eRfaA"
)

cipher = DES.new(key, DES.MODE_CBC, iv)
plaintext = unpad(cipher.decrypt(ciphertext), DES.block_size)
print(plaintext.decode())
```

输出为：

```text
0xGame{8edf2e65-1cb3-2e1a-b2d1-b54d3d4bddc5}
```

## 方法总结

逆向加密校验时不能只记录算法名称，还要逐项确认模式、填充、编码、Key 和 IV。此题最容易出错的地方是把异或后的字符串同时当作 Key 与 IV；实际程序只用它作 Key，IV 仍是资源中的原始 8 字节字符串。
