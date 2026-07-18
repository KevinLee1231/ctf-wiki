# week2Despacito

## 题目简述

程序完整实现了标准 DES。通过函数名、初始/逆置换表、密钥扩展和 16 轮分组处理可以识别算法；已知 8 字节密钥 `0xgame21` 与 `cipher.txt` 后，按 ECB 模式解密，再对原始明文字节计算 MD5 作为 flag 内容。

## 解题过程

与普通自定义逐字节校验不同，程序中出现了 DES 固定的初始置换 IP、逆置换 FP 和轮密钥相关表。两个置换表采用从 0 开始的下标，其开头分别为：

```text
IP_Table:   57, 49, 41, 33, 25, 17,  9,  1, ...
IP_1_Table: 39,  7, 47, 15, 55, 23, 63, 31, ...
```

这两组序列与标准 DES 的 IP、FP 完全对应。反编译出的文件处理主线可整理为下面的等价代码；截图中的红框只是在强调 `DES_MakeSubKeys` 和 `DES_Encrypt_Block` 两处调用，没有额外视觉信息：

```cpp
char8_to_bit64(key_bytes, key_bits);
DES_MakeSubKeys(key_bits, subkeys);

while ((size = fread(buffer, 1, 8, input)) == 8) {
    DES_Encrypt_Block(buffer, subkeys, output_block);
    fwrite(output_block, 1, 8, output);
}

if (size != 0) {
    memset(buffer + size, 0, 8 - size);
    DES_Encrypt_Block(buffer, subkeys, output_block);
    fwrite(output_block, 1, 8, output);
}
```

由此还能确认程序以 8 字节为一组读取文件，并对最后不足 8 字节的分组补零。

`DES.cpp` 还显示数据按 8 字节分组处理，密钥为恰好 8 字节的 `0xgame21`，没有 CBC 链式异或或 IV，因此模式为 ECB。`cipher.txt` 长度为 96 字节，正好是 8 的倍数；解密后的 96 字节就是要参与 MD5 的原始明文，不要自行 `strip()` 或删除末尾字节。

```python
import hashlib
from pathlib import Path
from Crypto.Cipher import DES

ciphertext = Path("cipher.txt").read_bytes()
key = b"0xgame21"

assert len(key) == 8
assert len(ciphertext) % 8 == 0

plaintext = DES.new(key, DES.MODE_ECB).decrypt(ciphertext)
print(plaintext.decode())
print(f"0xGame{{{hashlib.md5(plaintext).hexdigest()}}}")
```

明文为：

```text
wLpGWGNJYVvBwLBCVzgatsuGZaAzbBUHPXjoUqdahnPzeLdZrKntUcYwPHFHxtrVgzyWwdUtYvgiQuLyqwQPFVaWQLaGuupA
```

最终 flag：

```text
0xGame{83b9879f334340ef42dbb9f40468fc84}
```

## 方法总结

本题考察标准算法识别，而不是从头阅读每个轮函数。DES 的固定置换表、8 字节密钥、8 字节分组和 16 轮结构共同构成稳定指纹；识别后仍需确认具体模式、填充以及题目要求对“明文文本”还是“原始明文字节”计算摘要。
