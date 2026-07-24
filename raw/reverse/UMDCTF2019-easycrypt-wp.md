# UMDCTF 2019 - EasyCrypt

## 题目简述

题目提供加密程序和文件 `ENCRYPTED`。程序先对每个字节做可逆的半字节变换，再以 32 字节为单位、按伪随机顺序重排分块。随机过程只由文件长度决定，因此可以完整逆运算。

## 解题过程

单字节变换保留低四位，并依据最高位重组高半字节：

```python
def encrypt_byte(value: int) -> int:
    if value >= 0x80:
        return ((~(value * 2)) & 0xe0) | (value & 0x0f)
    return ((value * 2) & 0xe0) | 0x10 | (value & 0x0f)

inverse = {encrypt_byte(value): value for value in range(256)}
assert len(inverse) == 256
```

程序以密文长度调用 `srand`，取第一次 `rand()` 的低 16 位作为状态，后续使用：

```text
state = (state * 0x083d + 0x2439) & 0x7fff
```

它依次处理大小为 512、256、128、64、32 字节的块，并在每个块中排列 32 字节小块。解密时用同一文件长度复现随机序列，记录加密输出位置对应的原位置，先把小块放回，再查逆字节表：

```python
plaintext = bytes(inverse[value] for value in reordered_ciphertext)
open("decrypted.jpg", "wb").write(plaintext)
```

恢复文件具有正常 JPEG 头，尺寸为 2269×133。图中显示：

![逆转字节变换和分块排列后恢复的 flag 横幅](./UMDCTF2019-easycrypt-wp/decrypted-flag-banner.jpg)

```text
UMDCTF-{Th4t_wasnT_evEn_Crypt0}
```

其 SHA-256 与官方摘要一致。

## 方法总结

“使用随机数”不等于不可逆：种子若完全由公开的文件长度决定，排列就可以确定性重放。面对自定义加密，应把流程拆成字节代换和位置置换两层，分别证明可逆，再按加密的逆序恢复。输出文件的 magic、可视内容和官方摘要构成三重校验。
