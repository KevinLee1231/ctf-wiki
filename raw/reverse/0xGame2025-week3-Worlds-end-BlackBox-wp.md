# World's_end_BlackBox

## 题目简述

附件是带调试符号的 64 位 C++ 控制台程序。虽然 STL 模板让反编译结果显得冗长，但核心符号仍然保留：`KeyGenerate`、`rc4_ksa`、`rc4_prga`、`rc4_encrypt`，以及全局对象 `KeyEnc`、`DeKey`、`TrueKey` 和 `KALEIDXSCOPE`。

程序有两次输入。第一次要求 12 字符口令，正确值由 `KeyGenerate` 使用 XXTEA 在运行时解出；第二次要求 51 字符 flag，并用该口令执行改造 RC4。最终不是比较字符串，而是把加密结果转成整数数组，与全局的 51 项密文数组比较。

## 解题过程

### 1. 还原运行时口令

`main` 先检查第一次输入长度是否为 `0xC`，随后执行：

```cpp
TrueKey = KeyGenerate(KeyEnc, DeKey);
if (inputKey != TrueKey) {
    exit(0);
}
```

二进制中的两个全局常量为：

```text
KeyEnc = 2c26c188db0d0a4b70c3f226
DeKey  = 0123456789abcdef
```

`KeyGenerate` 要求密钥长度为 16，把 `KeyEnc` 每 8 个十六进制字符解析成一个 32 位整数，并按小端序把解密结果还原为字节。循环轮数、常量 `0x9E3779B9` 和 `MX` 结构都对应 XXTEA 解密。

下面的脚本可以静态还原第一次输入：

```python
import struct


MASK = 0xFFFFFFFF
DELTA = 0x9E3779B9


def mx(z, y, total, key, position, selector):
    return (
        (
            ((z >> 5) ^ ((y << 2) & MASK))
            + ((y >> 3) ^ ((z << 4) & MASK))
        )
        ^ (
            (total ^ y)
            + (key[(position & 3) ^ selector] ^ z)
        )
    ) & MASK


def xxtea_decrypt(words, key):
    values = words[:]
    count = len(values)
    rounds = 6 + 52 // count
    total = (rounds * DELTA) & MASK
    y = values[0]

    while rounds:
        selector = (total >> 2) & 3

        for position in range(count - 1, 0, -1):
            z = values[position - 1]
            values[position] = (
                values[position]
                - mx(z, y, total, key, position, selector)
            ) & MASK
            y = values[position]

        z = values[count - 1]
        values[0] = (
            values[0]
            - mx(z, y, total, key, 0, selector)
        ) & MASK
        y = values[0]
        total = (total - DELTA) & MASK
        rounds -= 1

    return values


encrypted_hex = "2c26c188db0d0a4b70c3f226"
encrypted_words = [
    int(encrypted_hex[index:index + 8], 16)
    for index in range(0, len(encrypted_hex), 8)
]
key_words = list(struct.unpack("<4I", b"0123456789abcdef"))
plain_words = xxtea_decrypt(encrypted_words, key_words)
runtime_key = b"".join(
    struct.pack("<I", word) for word in plain_words
).decode()

print(runtime_key)
```

输出：

```text
XaleidscopiX
```

动态调试到 `KeyGenerate` 返回后，也能在 `TrueKey` 对象中看到同一字符串，这可作为静态脚本结果的交叉验证。

### 2. 识别改造 RC4

通过第一阶段后，程序检查第二次输入长度是否为 `0x33`，即 51 字符，然后调用：

```cpp
encrypted = rc4_encrypt(flagInput, inputKey);
```

`rc4_ksa` 和 `rc4_prga` 都是标准 RC4：初始化 `S[i]=i`，按密钥交换 256 项状态，再逐字节产生密钥流。唯一改动出现在输出阶段：

```cpp
output[i] = input[i] ^ keyStream[i] ^ 7;
```

因此每字节满足 $C_i=P_i\oplus K_i\oplus7$。异或可逆，使用相同 RC4 密钥流再异或 `7` 就能从内置密文恢复明文。

### 3. 解密内置数组

从全局 `KALEIDXSCOPE` 取出的 51 项密文如下。这个名字只是变量名，实际内容不是字符串 `KALEIDXSCOPE`：

```python
ciphertext = bytes([
    0xFC, 0xEA, 0x15, 0x2C, 0x86, 0x38, 0x3F, 0xF3, 0x92, 0xCE,
    0xDA, 0x8E, 0x48, 0xD3, 0x07, 0x9F, 0xD9, 0x57, 0xB1, 0xEE,
    0x41, 0x9A, 0x4D, 0xC5, 0x65, 0x6A, 0xFF, 0xC9, 0x5D, 0x34,
    0xAD, 0xEA, 0xB1, 0x20, 0x4B, 0xDC, 0xBD, 0xD2, 0x35, 0x02,
    0x84, 0x35, 0x71, 0xEC, 0xE0, 0x48, 0x8E, 0xEA, 0x7B, 0xAA,
    0xCF,
])


def modified_rc4(data: bytes, key: bytes) -> bytes:
    state = list(range(256))
    j = 0

    for i in range(256):
        j = (j + state[i] + key[i % len(key)]) & 0xFF
        state[i], state[j] = state[j], state[i]

    i = 0
    j = 0
    output = bytearray()

    for value in data:
        i = (i + 1) & 0xFF
        j = (j + state[i]) & 0xFF
        state[i], state[j] = state[j], state[i]
        stream = state[(state[i] + state[j]) & 0xFF]
        output.append(value ^ stream ^ 7)

    return bytes(output)


flag = modified_rc4(ciphertext, b"XaleidscopiX").decode()
print(flag)
```

输出：

```text
0xGame{RC4_15_4_b4s1c&fl3x1bl3_3ncrYp710n4lg0r17hm}
```

## 方法总结

本题的完整链条是“XXTEA 解出运行时口令 → 输入口令通过比较 → 以口令作为 RC4 密钥 → 每字节额外异或 `7` → 与 51 项数组比较”。调试符号已经给出很强的算法线索，分析时应先过滤 STL 噪声，围绕具名函数和全局常量建立数据流。

动态调试可以直接截获 `TrueKey`，但静态还原能进一步说明口令从何而来。对于“魔改 RC4”，应分别确认 KSA、PRGA 和最终组合式；本题只修改了最后一个字节变换，因此无需重写状态机，保留标准 RC4 后补一个异或即可。
