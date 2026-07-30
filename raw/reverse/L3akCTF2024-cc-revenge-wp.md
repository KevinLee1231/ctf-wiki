# L3akCTF 2024 CC Revenge Writeup

## 题目简述

CC Revenge 延续了 CC 的 Schiffy128 自定义密码，但源码删除了解密函数，并修改了 S 盒初值、轮密钥掩码和轮函数。附件给出 32 字节 `flag.bin`，原始 128 位密钥仍然明文硬编码：

```text
deadbeef000000000000000badc0ffee
```

算法仍是 32 轮、左右各 64 位的 Feistel 网络，因此即使没有现成的 `decrypt`，也能根据加密状态转移直接构造逆过程。

## 解题过程

这一版与前题相比有三个关键变化：

```text
S 盒初值：        255
轮密钥异或常量：  0xdeeeef
轮函数附加常量：  0xdeadbeef00000000
```

S 盒仍按下式递推：

$$
S_0=255,\qquad S_i=(37S_{i-1}+9)\bmod 256.
$$

轮函数对输入半块执行：

```text
x ^= low64(round_key) ^ MAGIC
x  = bytewise_sbox(x)
x ^= high64(round_key) ^ MAGIC
```

加密每轮为：

```text
(left, right) = (right, left XOR F(right, round_key[i]))
```

所以从密文状态 $(L',R')$ 回退一轮时：

```text
right = left'
left  = right' XOR F(left', round_key[i])
```

完整脚本如下：

```python
from pathlib import Path

MASK64 = (1 << 64) - 1
MASK128 = (1 << 128) - 1
KEY = 0xdeadbeef000000000000000badc0ffee
MAGIC = 0xdeadbeef00000000


def rol128(value, count):
    count %= 128
    return (
        (value << count) | (value >> (128 - count))
    ) & MASK128


def make_sbox():
    table = [255]
    for _ in range(255):
        table.append((37 * table[-1] + 9) & 0xff)
    return table


def make_round_keys(key):
    result = []
    round_key = key

    for i in range(32):
        round_key = rol128(round_key, 7 * i) ^ 0xdeeeef
        result.append(round_key)

    return result


SBOX = make_sbox()
ROUND_KEYS = make_round_keys(KEY)


def feistel_function(block, round_key):
    block ^= (round_key & MASK64) ^ MAGIC

    substituted = bytes(
        SBOX[byte]
        for byte in block.to_bytes(8, "little")
    )
    block = int.from_bytes(substituted, "little")

    return block ^ (round_key >> 64) ^ MAGIC


def decrypt_block(ciphertext):
    left = ciphertext >> 64
    right = ciphertext & MASK64

    for i in range(31, -1, -1):
        left, right = (
            right ^ feistel_function(left, ROUND_KEYS[i]),
            left,
        )

    return ((left << 64) | right).to_bytes(16, "big")


data = Path("flag.bin").read_bytes()
plain = b"".join(
    decrypt_block(int.from_bytes(data[i:i + 16], "big"))
    for i in range(0, len(data), 16)
)

padding = plain[-1]
assert plain[-padding:] == bytes([padding]) * padding
print(plain[:-padding].decode())
```

运行得到：

```text
L3AK{R3venge_0f_Th3_Sch1ffy}
```

## 方法总结

- “删除解密函数”不会使 Feistel 网络变成单向结构；加密轮本身已经给出了唯一的逆状态转移。
- Revenge 题不能直接复用前题参数。S 盒初值、密钥掩码、高低 64 位使用顺序以及 `MAGIC` 都发生了变化。
- Rust 的 `to_le_bytes` 与最终 `to_be_bytes` 位于不同层次：前者只用于轮函数逐字节代换，后者决定密文文件中 128 位分组的顺序。
- 最终应检查 PKCS#7 填充，而不是看到可打印前缀后直接截断，以便发现端序或轮次实现错误。
