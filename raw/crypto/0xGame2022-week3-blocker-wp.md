# week3blocker

## 题目简述

题目实现了一个 4 字节分组的自定义密码。单块先与常量 $a$ 异或、循环左移 11 位、再与常量 $b$ 异或；消息层又把每个变换结果与前一密文块异或，初始向量为 ASCII 字节 `0xgm`。附件给出了 $a$、$b$ 和完整密文，目标是逐块逆变换。

## 解题过程

令 $R_{11}$ 表示 32 位循环左移 11 位，$C_{-1}=\texttt{bytes\_to\_long(b"0xgm")}$，则加密关系为：

$$
C_i=\bigl(R_{11}(P_i\oplus a)\oplus b\bigr)\oplus C_{i-1}.
$$

先消去链式异或和 $b$，再执行循环右移 11 位，最后异或 $a$：

$$
P_i=R^{-1}_{11}\bigl(C_i\oplus C_{i-1}\oplus b\bigr)\oplus a.
$$

在 32 位上，右移 11 位等价于左移 $32-11=21$ 位。`bytes_to_long` 使用大端序，因此恢复每块时也固定输出 4 字节大端序，避免 `long_to_bytes` 丢掉前导零。

```python
from Crypto.Util.number import bytes_to_long

a = 232825750
b = 1828860569
cipher = (
    b"\x9a\x5d\xec\x18\xd9\x98\x1d\x85\x0b\x7d\x56\xf0"
    b"\xc9\x98\x8d\x85\x22\x3c\xf4\x02\x2b\xa1\x6d\xe7"
    b"\xa0\xa6\x64\x4a\x8b\x93\x75\x3f\x0b\x8d\xf6\x32"
)

MASK = 0xFFFFFFFF
BLOCK_SIZE = 4


def rol32(value, shift):
    shift %= 32
    return ((value << shift) | (value >> (32 - shift))) & MASK


def decrypt_transformed(value):
    value ^= b
    value = rol32(value, 21)  # ROL 21 == ROR 11
    value ^= a
    return value.to_bytes(4, "big")


blocks = [
    bytes_to_long(cipher[i:i + BLOCK_SIZE])
    for i in range(0, len(cipher), BLOCK_SIZE)
]

previous = bytes_to_long(b"0xgm")
plain = bytearray()
for current in blocks:
    plain.extend(decrypt_transformed(current ^ previous))
    previous = current

print(bytes(plain).rstrip(b"\x00").decode())
```

输出：

```text
0xGame{n0w_y0u_kn0w_bl0ck|c1pher?}
```

## 方法总结

逆向自定义分组密码时应按加密顺序反向撤销每个可逆操作：异或的逆仍是异或，循环左移 11 位的逆是循环右移 11 位。消息层的前一块依赖也必须保留，解第 $i$ 块要使用原始 $C_{i-1}$，不能用已经恢复的明文替代。固定块宽和端序同样重要，否则前导零会造成后续块错位。
