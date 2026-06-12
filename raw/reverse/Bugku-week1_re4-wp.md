# week1_re4

## 题目简述

题目是 TEA 加密逆向。64 位 PE 读取 44 字符输入，按 6 组 `uint32` 对执行 32 轮 TEA 加密，再与 12 个硬编码 `uint32` 比较。密钥是 16 字节字符串 `do_you_know_tea?`，delta 为 TEA 标准常量 `0x9e3779b9`。

## 解题过程

### 关键观察

反编译中可见 TEA 特征：

```text
delta = 0x9e3779b9
rounds = 32
key = "do_you_know_tea?"
```

期望值：

```python
expected = [
    0xC44B21FD, 0xE2588BD5, 0xA1A5C799, 0xC25668B5,
    0x3EFD89A5, 0x48A07CF6, 0x35760179, 0x6396B9DF,
    0xA5AB7711, 0x86D60B59, 0x0D84BE79, 0xFDE98A90,
]
```

### 求解步骤

```python
import struct

key = struct.unpack("<4I", b"do_you_know_tea?")
delta = 0x9e3779b9

def decrypt(v0, v1):
    s = (delta * 32) & 0xffffffff
    for _ in range(32):
        v1 = (v1 - (((v0 << 4) + key[2]) ^ ((v0 >> 5) + key[3]) ^ (v0 + s))) & 0xffffffff
        v0 = (v0 - (((v1 << 4) + key[0]) ^ ((v1 >> 5) + key[1]) ^ (v1 + s))) & 0xffffffff
        s = (s - delta) & 0xffffffff
    return v0, v1

out = bytearray()
for i in range(0, len(expected), 2):
    out += struct.pack("<II", *decrypt(expected[i], expected[i + 1]))
print(out[:44].decode())
```

输出：

```text
0xGame{c9fc98e9-3b72-420f-9dc3-00a94d8b126d}
```

## 方法总结

- 核心技巧：识别 TEA 常量和 32 轮 Feistel 结构，按相反顺序解密。
- 识别信号：`0x9e3779b9`、4 个 32 bit key、`v0/v1` 交替更新是 TEA/XTEA 系列强特征。
- 复用要点：不同 TEA 变体的 key 下标和加减方向可能不同，必须按反编译代码逐项对应，不能直接套模板。
