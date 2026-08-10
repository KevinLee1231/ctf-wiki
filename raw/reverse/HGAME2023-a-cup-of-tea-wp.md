# a_cup_of_tea

## 题目简述

程序使用魔改 TEA 校验输入。整体轮函数仍是标准 TEA 结构，但轮常量从常见的 `0x9E3779B9` 改为 `0x543210DD`。输入的前 32 字节按四个 64 位块加密，结果与八个 32 位密文整数比较；末尾的 `k}` 没有参与 TEA，而是以小端整数 `0x7D6B` 直接附在密文数组末尾。

## 解题过程

### 识别 TEA 参数

主函数依次调用：

```c
tea(&input[0], key);
tea(&input[8], key);
tea(&input[16], key);
tea(&input[24], key);
```

`tea` 每次接收两个 `uint32_t`，循环 32 轮。反编译代码中的 `0x543210DD` 是魔改后的 `delta`，128 位密钥在 IDA 中被优化成一个 XMM 常量；将其重新定义为 `int[4]` 后可导出：

```text
0x12345678, 0x23456789, 0x34567890, 0x45678901
```

八个密文 word 为：

```text
0x2E63829D, 0xC14E400F, 0x9B39BFB9, 0x5A1F8B14,
0x61886DDE, 0x6565C6CF, 0x9F064F64, 0x236A43F6
```

其中反编译结果出现的负十进制数只是同一 32 位位模式的有符号显示。

### 逆向 32 轮运算

所有加减法都必须限制在无符号 32 位范围。TEA 解密从 $sum=-32\times delta\pmod{2^{32}}$ 开始，每轮先恢复 $v_1$，再恢复 $v_0$：

```python
import struct

MASK = 0xFFFFFFFF
DELTA = 0x543210DD
KEY = [0x12345678, 0x23456789, 0x34567890, 0x45678901]
CIPHERTEXT = [
    0x2E63829D,
    0xC14E400F,
    0x9B39BFB9,
    0x5A1F8B14,
    0x61886DDE,
    0x6565C6CF,
    0x9F064F64,
    0x236A43F6,
]


def decrypt_block(v0, v1):
    total = (-DELTA * 32) & MASK

    for _ in range(32):
        v1 = (
            v1
            - (((v0 << 4) + KEY[2]) ^ (v0 + total) ^ ((v0 >> 5) + KEY[3]))
        ) & MASK
        v0 = (
            v0
            - (((v1 << 4) + KEY[0]) ^ (v1 + total) ^ ((v1 >> 5) + KEY[1]))
        ) & MASK
        total = (total + DELTA) & MASK

    return v0, v1


plaintext = bytearray()
for index in range(0, len(CIPHERTEXT), 2):
    block = decrypt_block(CIPHERTEXT[index], CIPHERTEXT[index + 1])
    plaintext.extend(struct.pack("<II", *block))

# Buf2[8] = 0x7D6B，在小端内存中就是 b"k}"。
plaintext.extend(struct.pack("<H", 0x7D6B))
print(plaintext.decode())
```

运行得到：

```text
hgame{Tea_15_4_v3ry_h3a1thy_drlnk}
```

## 方法总结

- 核心技巧：根据两个 32 位分组、四个 32 位密钥、32 轮移位加法异或结构识别 TEA，再替换题目使用的魔改 `delta`。
- 易错点：所有中间值都要按 `uint32_t` 截断；密文和明文 word 均按小端序解释；末尾 `k}` 是未加密数据。
- 复用要点：编译器把 128 位密钥折叠成 XMM 常量时，可在 IDA 中恢复为 `int[4]`；对优化后的表达式要先还原括号和无符号语义，再照搬标准解密模板。
