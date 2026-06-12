# MyMnemonic

## 题目简述

题目给出的核心附件是一张图片，图片末尾区域隐藏了一段黑白格编码的二进制数据。按 10 像素步长扫描图片，蓝色通道为 `255` 记作 `0`，否则记作 `1`，可提取出 192 位二进制串。

题目目标不是直接从图片中读出 flag，而是恢复钱包种子。192 位数据不是完整助记词比特流，而是 BIP39 的初始熵 `ENT`；对中文 18 词助记词而言，还需要根据 `SHA256(ENT)` 追加 6 位 checksum，得到 198 位后再按 11 位一组映射到 2048 个中文助记词表，最后用 BIP39 的 PBKDF2-HMAC-SHA512 派生 seed。

## 解题过程

首先提取出图片附件后面有黑白编码的二进制数据，先提取出来

```
from PIL import Image
img = Image.open("ma.png").convert("RGB")
w,h = img.size
line = ""
for i in range(0,h,10):
    for j in range(0,w,10):
        pixel = img.getpixel((j,i))
        if pixel[2] == 255:
            line += "0"
        else:
            line += "1"
print(line)
#000101111010111000011110110111010011101010000101110010100111101110110110100000
0010011100001101100001110110011011000011101101010000101101111011101011011000010
01110110001111000000111010111110100
```

得到192位的一个二进制数据
根据题意需要我们恢复钱包的种子，那么首先需要拿到助记词。根据Bip39协议，每种语言有一个助记词对照表，
一共2048个单词，由于2^11 = 2048，因此2048个助记词可以表示11位二进制数据的所有情况。
例如，“我”是助记词表中的第二个，那么对应的数就是00000000010，以此类推。

### 恢复Checksum

根据 Bip39 协议，助记词由“初始熵 (ENT)”和“校验和 (CS)”组成。可以很明显的发现，现在的长度192并不能刚好
被11整除，是因为省去了6位的Checksum（校验码），因此这里给出的是助记词的初始熵（ENT）。
再计算一下校验码，为101011。
$$CheckSum = \text{Bits}(\text{SHA256}(ENT))[0 : 6]$$
也可以用 `bip39-standalone` 验证：工具识别出 `Event Count = 192`、`Entropy Type = binary`，追加校验位后得到 18 个 word index。

### 恢复助记词

得到完整的数据后，11位一组，转10进制，得到这些助记词的索引：
[189, 903, 1466, 936, 741, 494, 1744, 156, 432, 1894, 1565, 1346, 1783, 728, 630, 480, 943, 1323]
在题目的查询网站上查询，得到对应的中文助记词：
["纳", "百", "福", "财", "源", "似", "水", "而", "至", "走", "大", "运", "事", "业", "如", "日", "中", "天"]

### 恢复种子

得到助记词，使用Bip39协议恢复种子即可，默认参数，Salt使用无密语的"mnemonic"
$$\text{Seed} = \text{PBKDF2}(\text{HMAC-SHA512}, \text{Key}, \text{Salt}, \text{Iterations})$$

```
import hashlib
import binascii
import unicodedata
def solve(words):
    mnemonic_sentence = " ".join(words)
    normalized_mnemonic = unicodedata.normalize('NFKD', mnemonic_sentence)
    salt = "mnemonic"
    normalized_salt = unicodedata.normalize('NFKD', salt)
    seed_bytes = hashlib.pbkdf2_hmac(
        'sha512',
        normalized_mnemonic.encode('utf-8'),
        normalized_salt.encode('utf-8'),
        2048,
        64
    )

    seed_hex = binascii.hexlify(seed_bytes).decode('utf-8')
    return seed_hex
WORDS = [
    "纳", "百", "福", "财", "源", "似", "水", "而", "至",
    "走", "大", "运", "事", "业", "如", "日", "中", "天"
]
if __name__ == "__main__":
    print(f"{solve(WORDS)}")
```

得到种子：
7243a5d4e66d0a6f1d5d51d0ea287f185741a78d864cd3778c101fe0367244f5de33f0c567fe2ed90fbe8181cf8a0
957e921bb562300f1d4a51c740bb8b79669

## 方法总结

BIP39 相关题要先区分“熵”和“完整助记词索引流”。如果二进制长度不能被 11 整除，不一定是提取错误，可能是只给了 `ENT`，需要按 `CS = ENT / 32` 补 checksum。中文助记词同样是 2048 词表、11 位一词；恢复 seed 时还要注意 NFKD 规范化、salt 为 `mnemonic + passphrase`，无密语时就是 `mnemonic`。
