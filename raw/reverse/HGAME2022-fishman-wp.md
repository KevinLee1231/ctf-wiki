# fishman

## 题目简述

程序在初始化函数中展开了一套分组密码实现，并在校验函数中以 8 字节为一组处理 32 字节输入。密钥和四个 64 位密文常量都保存在二进制中，需要先识别算法，再处理实现与常用密码库之间的字节序差异。

## 解题过程

从 `init` 和 `check` 两个函数入手。初始化代码中能找到 `0xD1310BA6` 等 Blowfish 常量，校验逻辑也以 Blowfish 的 64 位分组大小逐块处理数据，因此可以确认算法。继续跟踪密钥引用，得到：

```text
LET_U_D
```

程序保存的四个有符号 `long long` 密文为：

```c
long long data[4] = {
    -5409505419495256385LL,
    1428749241468231806LL,
    6435326525834898959LL,
    2019834963917240364LL
};
```

原程序在小端机器上把每个 8 字节分组拆成两个 32 位整数传给 `Blowfish_Decrypt`。PyCryptodome 的外部块表示采用大端序，所以不能直接对 `struct.pack("<q", value)` 的结果解密；要在解密前后分别翻转每个 4 字节半块：

```python
import struct

from Crypto.Cipher import Blowfish

values = [
    -5409505419495256385,
    1428749241468231806,
    6435326525834898959,
    2019834963917240364,
]

little_endian = b"".join(struct.pack("<q", value) for value in values)

def swap_word_endian(data):
    result = bytearray()
    for offset in range(0, len(data), 8):
        block = data[offset:offset + 8]
        result += block[:4][::-1]
        result += block[4:][::-1]
    return bytes(result)

library_input = swap_word_endian(little_endian)
decrypted = Blowfish.new(b"LET_U_D", Blowfish.MODE_ECB).decrypt(library_input)
plaintext = swap_word_endian(decrypted)
print(plaintext.decode())
```

输出为：

```text
hgame{D0_y0u_re411V_11k3_9Vthon}
```

## 方法总结

Blowfish 的固定初始化常量和 8 字节分组是本题最醒目的识别特征。恢复密钥与密文后，剩余风险在数据表示：反编译代码按本机小端 32 位整数操作左右半块，而通用密码库按网络序解释外部字节串。若直接调用库函数只会得到乱码，必须先对照原实现确认每个 32 位字的端序。
