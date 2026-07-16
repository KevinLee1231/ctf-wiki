# 我叫曼波

## 题目简述

服务每次只编码 flag 的一个字符，并同时允许查询本轮随机 RC4 密钥和密文。编码链依次为“RC4 → Base64 → 每个字符转 5 位三进制 → `0/1/2` 映射为 `曼波/哦耶/哇嗷`”，逐层逆变换即可恢复全部 96 个字符。

## 解题过程

服务端每次执行：

```python
flag_part = flag[i:i + 1]
key = str(random.randint(1, 1000))
ciphertext = RC4(flag_part, key)
base3_text = base3(ciphertext)
result = manbo_encode(base3_text)
```

自定义映射是：

```text
曼波 -> 0
哦耶 -> 1
哇嗷 -> 2
```

每个映射词恰好占两个 Unicode 字符。先每两个字符还原一个三进制位，再每 5 位转成一个 ASCII 字符，得到 Base64 文本；Base64 解码后再用同一 RC4 密钥异或即可得到当前 flag 字符。

完整交互脚本如下，主机和端口替换为当前实例：

```python
import base64
from pwn import remote


def rc4(text, key):
    state = list(range(256))
    key_stream = [key[i % len(key)] for i in range(256)]

    j = 0
    for i in range(256):
        j = (j + state[i] + ord(key_stream[i])) % 256
        state[i], state[j] = state[j], state[i]

    i = j = 0
    result = []
    for char in text:
        i = (i + 1) % 256
        j = (j + state[i]) % 256
        state[i], state[j] = state[j], state[i]
        value = state[(state[i] + state[j]) % 256]
        result.append(chr(ord(char) ^ value))
    return "".join(result)


def decode(key, manbo_text):
    table = {"曼波": "0", "哦耶": "1", "哇嗷": "2"}
    ternary = "".join(
        table[manbo_text[i:i + 2]]
        for i in range(0, len(manbo_text), 2)
    )
    b64_text = "".join(
        chr(int(ternary[i:i + 5], 3))
        for i in range(0, len(ternary), 5)
    )
    encrypted_char = base64.b64decode(b64_text).decode()
    return rc4(encrypted_char, key)


io = remote("HOST", PORT)
flag = []

for _ in range(96):
    io.sendlineafter(b"> ", b"1")

    io.sendlineafter(b"> ", b"2")
    key = io.recvline().strip().decode()

    io.sendlineafter(b"> ", b"3")
    ciphertext = io.recvline().strip().decode()

    flag.append(decode(key, ciphertext))
    print("".join(flag))

io.close()
```

最终得到：

```text
0xGame{OH_yEah_Wow_Duang_HajiMi_u_MADE_it!_and_MaY_5e_Y0u_hAv4_HeArD_7he_ST0ry_0f_Gu_Gao_MaN_B0}
```

## 方法总结

这道题没有要求破解 RC4 密钥：服务会主动泄露每一轮的密钥。真正工作是严格按相反顺序拆除自定义词表、三进制、Base64 和 RC4 四层编码，并注意原实现对 RC4 字符串执行 UTF-8 Base64 编解码。
