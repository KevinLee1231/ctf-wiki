# week3tls_rc4

## 题目简述

这是一个 Windows PE 逆向题。程序把部分逻辑放在 TLS callback 中；TLS callback 会在正常入口点和 `main` 之前执行，因此只从 `main` 开始分析会漏掉关键的流加密状态处理。

最终比较数据经历了 Base64、与循环密钥 `0xgame` 的逐字节异或，以及一段 RC4 结构的变换。目标是按相反顺序还原明文。

## 解题过程

从比较逻辑提取 40 字节常量和六字节密钥 `0xgame`。逆向时依次执行：

1. 对常量应用题目中的 RC4 结构；
2. 与循环密钥 `0xgame` 异或；
3. 对所得 ASCII 字符串做 Base64 解码。

题目实现与标准 RC4 有一个关键差异：KSA 结束后只把 `i` 清零，`j` 保留 KSA 的末值并直接进入 PRGA。若按标准实现把 `j` 重置为 0，结果不会形成合法 Base64。

```python
from base64 import b64decode

result = bytes([
    0xBA, 0xC5, 0x87, 0x89, 0x53, 0x15, 0x1B, 0x44,
    0x13, 0xFA, 0xD5, 0xBD, 0x48, 0xEA, 0xEE, 0x70,
    0x81, 0xD0, 0x18, 0xD6, 0x3B, 0x1E, 0x7F, 0xC2,
    0x7A, 0xE4, 0x17, 0xFD, 0x78, 0xA6, 0x01, 0xAF,
    0x5F, 0x3B, 0x98, 0x6A, 0xEA, 0xA9, 0x97, 0xE9,
])
key = b"0xgame"


def rc4_variant(data: bytes, key: bytes) -> bytes:
    box = list(range(256))
    j = 0

    # KSA
    for i in range(256):
        j = (j + box[i] + key[i % len(key)]) % 256
        box[i], box[j] = box[j], box[i]

    # 与题目一致：i 清零，但 j 延续 KSA 的末值。
    i = 0
    output = bytearray(data)
    for index in range(len(output)):
        i = (i + 1) % 256
        j = (j + box[i]) % 256
        box[i], box[j] = box[j], box[i]
        stream_byte = box[(box[i] + box[j]) % 256]
        output[index] ^= stream_byte

    return bytes(output)


stage1 = rc4_variant(result, key)
encoded = bytes(
    value ^ key[index % len(key)]
    for index, value in enumerate(stage1)
)

print(encoded.decode())
print(b64decode(encoded, validate=True).decode())
```

运行结果：

```text
MHhnYW1le3Qxc18xc19mMXI1dF90aGEwX21haW59
0xgame{t1s_1s_f1r5t_tha0_main}
```

## 方法总结

TLS callback 是 PE 正常执行链的一部分，逆向时应在入口点之外检查 TLS Directory。算法识别也不能只凭“结构像 RC4”就替换成库函数；本题保留 KSA 末尾 `j` 的细节会改变整条密钥流，必须逐条复现实际状态更新。
