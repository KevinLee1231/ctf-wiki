# re300.exe

## 题目简述

题目名 `MaybeNotStandrad` 暗示“不是标准编码”。附件是 32 位 Windows PE，程序读取 45 字符输入，将其用一张运行时生成的非标准 Base64 字母表编码，得到 60 字符后与硬编码密文比较。

核心机制与标准 Base64 相同，区别只在字母表被按 4 字节分组重排。还原字母表后对密文解码即可得到 flag。

## 解题过程

### 关键观察

关键字符串：

```text
Please input your flag:
Wow, you get it!
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/
ZnwgZ1sVf3VcQn8vX1RgPGRgfmRcVGFjPWVcdABcfELydGVhdF8QZWNzP1R8
```

校验函数要求 `strlen(input) == 45`，编码结果长度为 `60`，没有 padding。自定义字母表生成规则：

```text
标准表前 32 字节：每 4 字节按 [3,1,2,0] 重排
标准表后 32 字节：每 4 字节按 [2,0,3,1] 重排
```

### 求解步骤

构造自定义字母表并按 Base64 反解：

```python
STANDARD = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"

def build_table():
    custom = [""] * 64
    for i in range(0, 32, 4):
        custom[i + 0] = STANDARD[i + 3]
        custom[i + 1] = STANDARD[i + 1]
        custom[i + 2] = STANDARD[i + 2]
        custom[i + 3] = STANDARD[i + 0]
    for i in range(0, 32, 4):
        custom[32 + i + 0] = STANDARD[32 + i + 2]
        custom[32 + i + 1] = STANDARD[32 + i + 0]
        custom[32 + i + 2] = STANDARD[32 + i + 3]
        custom[32 + i + 3] = STANDARD[32 + i + 1]
    return "".join(custom)

def decode(s, table):
    lookup = {c: i for i, c in enumerate(table)}
    acc = bits = 0
    out = bytearray()
    for ch in s.rstrip("="):
        acc = (acc << 6) | lookup[ch]
        bits += 6
        if bits >= 8:
            bits -= 8
            out.append((acc >> bits) & 0xff)
    return bytes(out)

enc = "ZnwgZ1sVf3VcQn8vX1RgPGRgfmRcVGFjPWVcdABcfELydGVhdF8QZWNzP1R8"
print(decode(enc, build_table()).decode())
```

输出：

```text
flag{Use_NonSta0darD_Tab1e_t0_pr0tect_Secr3t}
```

## 方法总结

- 核心技巧：识别“标准算法 + 非标准字母表”的 Base64 变种。
- 识别信号：输入长度 45、输出长度 60、3 字节到 4 字符逻辑和 64 字符表同时出现，基本可判定 Base64 家族。
- 复用要点：不要直接套标准 Base64 解码；先还原查表顺序，再按自定义表构造逆映射。
