# week3_re2

## 题目简述

题目是 UPX 脱壳后的 XTEA 变体解密。原始 64 位 PE 被 UPX 加壳，脱壳后程序读取 32 字符输入，分成 4 个 64 bit block，每个 block 按 48 轮 XTEA 风格算法加密，再与硬编码密文比较。

## 解题过程

### 关键观察

文件段名 `UPX0/UPX1/UPX2` 暴露加壳。脱壳：

```bash
upx -d re2.exe -o re2_unpacked.exe
```

脱壳后主函数逻辑：

```text
strlen(input) == 32
key = [0x1234567, 0x89abcdef, 0xfedcba98, 0x76543210]
rounds = 48
delta = 0x9e3779b9
4 组 uint32 对加密后与 expected 比较
```

加密不是标准 TEA，而是 XTEA 风格：每轮的 key 下标依赖 `sum & 3` 和 `(sum >> 11) & 3`。

### 求解步骤

预计算每轮使用的 `k0/k1`，再倒序解密：

```python
import struct

expected_raw = bytes.fromhex(
    "4e97cd000d1fee9738364360bfcdbb52"
    "52530976dab834d236db99b5c245eefe"
)
expected = [struct.unpack_from("<I", expected_raw, i * 4)[0] for i in range(8)]

key = [0x1234567, 0x89abcdef, 0xfedcba98, 0x76543210]
delta = 0x9e3779b9

def decrypt(v0, v1):
    k0, k1 = [], []
    s = 0
    for _ in range(48):
        k0.append((key[s & 3] + s) & 0xffffffff)
        s = (s + delta) & 0xffffffff
        k1.append((s + key[(s >> 11) & 3]) & 0xffffffff)
    for i in range(47, -1, -1):
        v1 = (v1 - (k1[i] ^ (((v0 * 16) ^ (v0 >> 5)) + v0))) & 0xffffffff
        v0 = (v0 - (k0[i] ^ (((v1 << 4) ^ (v1 >> 5)) + v1))) & 0xffffffff
    return v0, v1

out = bytearray()
for i in range(0, 8, 2):
    out += struct.pack("<II", *decrypt(expected[i], expected[i + 1]))
print(out.decode())
```

输出：

```text
flag{could_you_decrypt_the_xtea}
```

## 方法总结

- 核心技巧：先 UPX 脱壳，再按 XTEA key schedule 反向解密。
- 识别信号：`0x9e3779b9`、多轮 `sum`、`sum & 3`、`sum >> 11` 是 XTEA 系列典型特征。
- 复用要点：TEA/XTEA 变体差异集中在 key 下标和更新顺序；写解密脚本时最好先正向复算一组 block 验证。
