# L3akCTF 2024 CC Writeup

## 题目简述

附件包含 Rust 编写的自定义 128 位分组密码 Schiffy128，以及 32 字节的 `flag.bin`。密钥直接硬编码为：

```text
deadbeef000000000000000badc0ffee
```

Schiffy128 本质上是 32 轮 Feistel 网络。每个 128 位分组拆成左右两个 64 位半块；轮函数先异或轮密钥高 64 位，逐字节过一个自定义 S 盒，再异或轮密钥低 64 位。由于 Feistel 网络不要求轮函数可逆，只需倒序使用轮密钥并交换更新方向即可解密。

## 解题过程

S 盒从 170 开始递推：

$$
S_0=170,\qquad S_i=(37S_{i-1}+9)\bmod 256.
$$

轮密钥由 128 位循环左移与常量 `0xabcdef` 生成。第 0 轮相当于原始密钥异或该常量，之后第 $i$ 轮把上一轮密钥左旋 $7i$ 位再异或常量。

加密轮为：

```text
new_left  = right
new_right = left XOR F(right, round_key[i])
```

因此解密时从第 31 轮倒推：

```text
new_right = left
new_left  = right XOR F(left, round_key[i])
```

附件按大端序写入每个 128 位密文块，明文使用 PKCS#7 填充。完整解密脚本如下：

```python
from pathlib import Path

MASK64 = (1 << 64) - 1
MASK128 = (1 << 128) - 1
KEY = 0xdeadbeef000000000000000badc0ffee


def rol128(value, count):
    count %= 128
    return (
        (value << count) | (value >> (128 - count))
    ) & MASK128


def make_sbox():
    table = [170]
    for _ in range(255):
        table.append((37 * table[-1] + 9) & 0xff)
    return table


def make_round_keys(key):
    result = []
    round_key = key

    for i in range(32):
        round_key = rol128(round_key, 7 * i) ^ 0xabcdef
        result.append(round_key)

    return result


SBOX = make_sbox()
ROUND_KEYS = make_round_keys(KEY)


def feistel_function(block, round_key):
    block ^= round_key >> 64

    substituted = bytes(
        SBOX[byte]
        for byte in block.to_bytes(8, "little")
    )
    block = int.from_bytes(substituted, "little")

    return block ^ (round_key & MASK64)


def decrypt_block(ciphertext):
    left = ciphertext >> 64
    right = ciphertext & MASK64

    for i in range(31, -1, -1):
        left, right = (
            right ^ feistel_function(left, ROUND_KEYS[i]),
            left,
        )

    plaintext = (left << 64) | right
    return plaintext.to_bytes(16, "big")


data = Path("flag.bin").read_bytes()
plain = b"".join(
    decrypt_block(int.from_bytes(data[i:i + 16], "big"))
    for i in range(0, len(data), 16)
)

padding = plain[-1]
assert plain[-padding:] == bytes([padding]) * padding
print(plain[:-padding].decode())
```

运行结果为：

```text
L3AK{its_all_started_with_C}
```

仓库内官方脚本的输入路径写成了 `../dev/ciphertext.bin`，但实际应读取分发目录中的 `flag.bin`；`solution/ciphertext.bin` 与它的 SHA-256 完全相同，也可作为等价输入。

## 方法总结

- 识别出 Feistel 结构后，轮函数本身是否可逆并不重要；解密只需倒序轮密钥并按逆向状态更新。
- 需要分别核对 128 位分组端序、64 位半块顺序和轮函数内部的字节序。这里密文块是大端，S 盒处理使用 Rust 的小端字节数组。
- 自定义密码题不要只看算法名称。轮密钥生成、S 盒递推、轮数和最终填充都必须从实现中逐项复现。
- 官方脚本出现路径笔误时，应以文件哈希和实际分发物为依据确认输入，不能据此误判题目材料缺失。
