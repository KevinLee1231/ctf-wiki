# APT213 - Our saviour

## 题目简述

题目给出被 RApTor 加密的 `flag.jpg.nxsfx`，要求结合前两阶段恢复文件。0x2 已经还原恶意程序的混合加密格式，0x3 则能从 C2 数据库中取得受害主机 `SNAIL-CORP` 对应的 RSA 私钥。

决定性步骤是按样本定义拆分“RSA-OAEP 包装的随机 AES 密钥”和“AES-CBC 加密的文件内容”，再执行正确的去填充，因此归入密码方向。

## 解题过程

RApTor 每次加密文件时生成随机 16 字节 AES 密钥，使用全零的 16 字节 IV 和 AES-CBC 加密 PKCS#7 填充后的原文件；随后用受害主机的 RSA 公钥通过 OAEP 包装 AES 密钥。写入格式为：

```text
N0X1I0USF0X || AES-CBC ciphertext || RSA-OAEP(AES key) || uint16_be(key_blob_len)
```

其中 `N0X1I0USF0X` 是 11 字节魔数，最后两个字节以大端序记录 RSA 密钥块长度。样本的总长度是 132445 字节，尾部长度字段为 `0x0100`，即 256 字节；去掉魔数、RSA 块和长度字段后，AES 密文为 132176 字节，恰好是 16 的倍数。

在 0x3 的 C2 数据库中执行：

```sql
SELECT privkey
FROM hosts
WHERE hostname = 'SNAIL-CORP';
```

将返回的 PEM 保存为 `john-private.pem`。完整解密脚本如下：

```python
from pathlib import Path
import struct

from Crypto.Cipher import AES, PKCS1_OAEP
from Crypto.PublicKey import RSA


MAGIC = b"N0X1I0USF0X"
encrypted = Path("flag.jpg.nxsfx").read_bytes()
private_key = RSA.import_key(Path("john-private.pem").read_bytes())

assert encrypted.startswith(MAGIC)

wrapped_length = struct.unpack(">H", encrypted[-2:])[0]
wrapped_key = encrypted[-2 - wrapped_length:-2]
ciphertext = encrypted[len(MAGIC):-2 - wrapped_length]

assert wrapped_length == private_key.size_in_bytes()
assert len(ciphertext) % AES.block_size == 0

aes_key = PKCS1_OAEP.new(private_key).decrypt(wrapped_key)
plaintext = AES.new(
    aes_key,
    AES.MODE_CBC,
    iv=b"\x00" * AES.block_size,
).decrypt(ciphertext)

padding_length = plaintext[-1]
assert 1 <= padding_length <= AES.block_size
assert plaintext.endswith(bytes([padding_length]) * padding_length)
plaintext = plaintext[:-padding_length]

assert plaintext.startswith(b"\xff\xd8\xff")
assert plaintext.endswith(b"\xff\xd9")
Path("flag.jpg").write_bytes(plaintext)
```

OAEP 解开 256 字节 RSA 块后得到 16 字节 AES 密钥；CBC 解密和严格 PKCS#7 去填充后，输出为 132168 字节的有效 JPEG，SHA-256 为：

```text
d9f4b4f91c9d9a5f8069cb6c5dc6b2810217f199da454320d3d2f6d764f2ff5b
```

打开解密后的 JPEG 后，图像中央给出的 flag 为：

```text
shellmates{wh3r3_d1d_th3_cr1m1n4l_g0?_h3_f4il3ds0mw4re!}
```

## 方法总结

本题需要先按文件尾部解析变长的 RSA 块，而不能凭 2048 位密钥直接硬编码偏移；尾部大端长度字段提供了稳定边界。剩余中间区域才是 AES-CBC 密文，开头 11 字节魔数既用于识别文件，也用于避免重复加密。

混合加密本身没有被破解：恢复成立是因为上一阶段泄露了与受害主机公钥配对的 RSA 私钥。验证应同时覆盖 RSA 块长度、AES 分组对齐、PKCS#7 每个填充字节以及 JPEG 首尾魔数；只看到能打开的部分图像不足以证明所有边界都解析正确。
