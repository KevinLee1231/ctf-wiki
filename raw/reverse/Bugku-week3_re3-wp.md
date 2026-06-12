# week3_re3

## 题目简述

题目是 64 位 PE RC4 逆向。程序读取 42 字符输入，用 RC4 加密后与 42 字节硬编码密文比较。S-box 初始化函数使用 key `0xGame2022`，因此用同一 key 对密文再跑 RC4 即可得到 flag。

## 解题过程

### 关键观察

主函数核心：

```text
read input length 0x2a
FUN_0040170d(0x408040, input, 42)
memcmp(input, DAT_00404040, 42)
```

`FUN_0040170d` 是标准 RC4 PRGA：

```c
i = (i + 1) % 256;
j = (j + sbox[i]) % 256;
swap(sbox[i], sbox[j]);
input[k] ^= sbox[(sbox[i] + sbox[j]) % 256];
```

S-box 初始化处：

```text
FUN_00401550(0x408040, "0xGame2022", 10)
```

### 求解步骤

```python
ct = bytes([
    0xdd,0x84,0x73,0x53,0xec,0x7c,0x5c,0xd4,0xe6,0x5e,
    0xe2,0x43,0x5f,0x39,0xe3,0x6e,0x5e,0x8e,0x11,0x97,
    0xfc,0x24,0x49,0x60,0x77,0x29,0x10,0xda,0x23,0x4d,
    0x3d,0x38,0x0a,0x76,0xb6,0x08,0xc0,0x38,0x91,0x28,
    0x43,0x91,
])

def rc4(data, key):
    s = list(range(256))
    j = 0
    for i in range(256):
        j = (j + s[i] + key[i % len(key)]) & 0xff
        s[i], s[j] = s[j], s[i]
    i = j = 0
    out = bytearray()
    for b in data:
        i = (i + 1) & 0xff
        j = (j + s[i]) & 0xff
        s[i], s[j] = s[j], s[i]
        out.append(b ^ s[(s[i] + s[j]) & 0xff])
    return bytes(out)

print(rc4(ct, b"0xGame2022").decode())
```

输出：

```text
flag{bc3ed83b-ffe3-4122-8fbe-651b1a591da7}
```

## 方法总结

- 核心技巧：识别 RC4 KSA/PRGA 并利用流密码加解密对称性。
- 识别信号：256 长度 S-box、双索引 `i/j`、swap 和 `s[(s[i]+s[j])&0xff]` 是 RC4 强特征。
- 复用要点：密钥长度要从初始化调用参数确认；本题 key 是 10 字节 `0xGame2022`，不是字符串附近其它干扰常量。
