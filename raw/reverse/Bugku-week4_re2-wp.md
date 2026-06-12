# week4_re2

## 题目简述

题目是 AES-128-ECB 逆向。64 位 ELF 读取 42 字符输入，按 3 个 AES block 加密后与 48 字节硬编码密文比较。密钥为 `00 01 02 ... 0f`，第三个 block 以零填充。

## 解题过程

### 关键观察

反编译主函数可见：

```text
FUN_00102121(0x10)                 # AES-128, Nk=4, Nr=10
FUN_00101de5(key, round_keys)       # key expansion
FUN_001021a1(input+0,  out+0)
FUN_001021a1(input+16, out+16)
FUN_001021a1(input+32, out+32)
memcmp(out, expected, 48)
```

密钥取主函数中 32 字节常量的前 16 字节：

```text
00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f
```

### 求解步骤

```python
from Crypto.Cipher import AES

key = bytes(range(16))
ct = bytes.fromhex(
    "1892f613761123eff70ab761461610c8"
    "4da194c18a2e8fba568810956edf74ee"
    "55812d305905ed63d27c1347a9a3f620"
)

pt = AES.new(key, AES.MODE_ECB).decrypt(ct)
print(pt.rstrip(b"\x00").decode())
```

输出：

```text
flag{2f2d0017-80f6-4f6f-97c9-bc4e9b21f3b1}
```

## 方法总结

- 核心技巧：识别 AES key schedule 和 block encrypt，直接用提取的密钥 ECB 解密。
- 识别信号：AES S-box、Nk/Nr 分支、SubBytes/ShiftRows/MixColumns/AddRoundKey 调用链是 AES 的强特征。
- 复用要点：题目输入 42 字节但比较 48 字节，说明最后一个 block 有填充或零扩展；解密后要去掉尾部空字节。
