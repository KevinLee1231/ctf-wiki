# TELF

## 题目简述

附件是一个经过 UPX 压缩的 64 位 ELF，但文件偏移 `0xEC` 处的标识被从 `55 50 58 21`（`UPX!`）改成了 `58 31 63 21`（`X1c!`），导致 UPX 拒绝识别和脱壳。

损坏值与标准值的差异如下：

| 文件偏移 | 题目附件 | 标准 UPX |
| --- | --- | --- |
| `0xEC`～`0xEF` | `58 31 63 21`（`X1c!`） | `55 50 58 21`（`UPX!`） |

恢复标识并脱壳后，程序使用 `srand(1010000)` 和四次 `rand()` 生成 TEA 密钥，再把 56 字节输入按 8 字节小端分组加密，与内置密文比较。

## 解题过程

先复制附件，只修改副本中 `0xEC` 开始的前三个字节；末尾的 `21`（`!`）本来就是正确的：

```bash
cp TELF TELF.fixed
printf '\x55\x50\x58' | dd of=TELF.fixed bs=1 seek=$((0xEC)) conv=notrunc
upx -d -o TELF.unpacked TELF.fixed
```

脱壳后的关键密钥生成代码为：

```c
srand(1010000);
uint32_t key[4];
for (int i = 0; i < 4; i++)
    key[i] = (uint32_t)rand();
```

在题目运行环境的 Linux glibc 中得到：

```text
7E4D087B 7A4DB733 70FE9DF0 595607F7
```

同一个种子不保证在 Windows C 运行库中产生相同序列，因此不能用 Windows 的 `rand()` 结果代替。得到密钥后，对七个 8 字节分组执行 32 轮标准 TEA 逆运算。源码使用 `memcpy` 把字节复制到 `uint32_t[2]`，在 x86-64 上按小端序解释。

```python
import struct

MASK = 0xFFFFFFFF
DELTA = 0x9E3779B9
KEY = (0x7E4D087B, 0x7A4DB733, 0x70FE9DF0, 0x595607F7)

CIPHERTEXT = bytes.fromhex(
    "AD DA 01 DC AE 5B 8A 08 4E F5 4F 8F 6E 5F 9D 9E "
    "0A 4E A9 08 25 AB 45 C2 4B C9 8F 43 3D 51 D6 28 "
    "F6 72 CD F4 2B B4 4A 3B FB 36 66 EF D6 8A 8C B2 "
    "EB 1A 9C 1B 0A 9C 1F 53"
)


def decrypt_block(block: bytes) -> bytes:
    v0, v1 = struct.unpack("<2I", block)
    total = (DELTA * 32) & MASK

    for _ in range(32):
        mix1 = (
            (((v0 << 4) + KEY[2]) & MASK)
            ^ ((v0 + total) & MASK)
            ^ (((v0 >> 5) + KEY[3]) & MASK)
        )
        v1 = (v1 - mix1) & MASK

        mix0 = (
            (((v1 << 4) + KEY[0]) & MASK)
            ^ ((v1 + total) & MASK)
            ^ (((v1 >> 5) + KEY[1]) & MASK)
        )
        v0 = (v0 - mix0) & MASK
        total = (total - DELTA) & MASK

    return struct.pack("<2I", v0, v1)


plaintext = b"".join(
    decrypt_block(CIPHERTEXT[offset:offset + 8])
    for offset in range(0, len(CIPHERTEXT), 8)
)
print(plaintext.decode())
```

输出为：

```text
0xGame{PANDORA-PANRADOXXX-101AP-9CDE02B83F5D6-7B1A9C348}
```

## 方法总结

本题有两层：先修复 `0xEC` 处的 UPX 魔改标识，再恢复 Linux glibc 随机序列和 TEA。`X1c! → UPX!`、种子 `1010000`、四个密钥、56 字节密文、32 轮和小端分组都是复现所需的关键事实。动态调试并非唯一方案，只要在正确运行库中生成密钥，便可离线完成解密。
