# Rustdroid

## 题目简述

题目是 Android APK，Java 层获取输入后进入 native `check`。native 层判断长度为 43，且以 `WMCTF{` 开头、以 `}` 结尾。中间 36 字节先做单字节变换，再进入 RC4，最后与固定密文比较。

## 解题过程

### 关键机制

单字节变换：

```python
def single_byte_encrypt(x):
    result = x
    result = (result >> 1) | ((result << 7) & 0xff)
    result ^= 0xef
    result = (result >> 2) | ((result << 6) & 0xff)
    result ^= 0xbe
    result = (result >> 3) | ((result << 5) & 0xff)
    result ^= 0xad
    result = (result >> 4) | ((result << 4) & 0xff)
    result ^= 0xde
    result = (result >> 5) | ((result << 3) & 0xff)
    return result
```

RC4 参数：

```python
encode = [
    0x1F, 0xBA, 0x15, 0x42, 0x59, 0xCE, 0x4F, 0x4E,
    0x94, 0xD9, 0xBF, 0x69, 0xAE, 0x5B, 0x74, 0x0C,
    0xC0, 0xFC, 0x8A, 0x7F, 0x9C, 0x1E, 0x08, 0x87,
    0xF5, 0x6B, 0x64, 0xF5, 0x87, 0x8F, 0xB0, 0x2B,
    0xE2, 0x53, 0xFF, 0x29
]
key = [0x66, 0x75, 0x6E, 0x40, 0x65, 0x5A]  # fun@eZ
xor_table = [0x77, 0x88, 0x99, 0x66]
```

RC4 输出时额外与 `xor_table[index % 4]` 异或。

### 求解步骤

先对密文做 RC4 逆运算，得到单字节变换后的目标数组：

```python
def rc4(key, data):
    s = list(range(256))
    j = 0
    for i in range(256):
        j = (j + s[i] + key[i % len(key)]) % 256
        s[i], s[j] = s[j], s[i]

    out = []
    i = j = 0
    for idx, y in enumerate(data):
        i = (i + 1) % 256
        j = (j + s[i]) % 256
        s[i], s[j] = s[j], s[i]
        k = s[(s[i] + s[j]) % 256]
        out.append(y ^ k ^ xor_table[idx % 4])
    return out
```

再枚举可打印字符，找出单字节变换后匹配的位置：

```python
decrypted_data = rc4(key, encode)
print("WMCTF{", end="")
for i in range(36):
    for ch in range(30, 128):
        if single_byte_encrypt(ch) == decrypted_data[i]:
            print(chr(ch), end="")
            break
print("}", end="")
```

得到：

```text
WMCTF{2a04aed7-e736-43c4-80a7-f6ed28de34eb}
```

## 方法总结

- 先 RC4 反推中间密文，再爆破单字节变换比直接逆旋转更简单。
- 长度 43 表明正文部分正好 36 字节：`len("WMCTF{}")=7`。
- 这类 native 校验题优先把格式检查、固定密文、密钥表和单字节变换拆开，再逐层逆回去。
