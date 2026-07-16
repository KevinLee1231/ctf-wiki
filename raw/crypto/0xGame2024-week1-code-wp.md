# code

## 题目简述

题目把 flag 等分为四段，并分别用四种 Python 数据表示输出：原始字节串、由 `bytes_to_long()` 得到的大端整数、十六进制字符串和 Base64 字节串。目标是按各自的逆变换恢复四段，再按原顺序拼接。

```text
m0 = b'0xGame{73d7'
m1 = 60928972245886112747629873
m2 = 3165662d393339332d3034
m3 = b'N2YwZTdjNGRlMX0='
```

## 解题过程

- `m0` 已经是字节串，直接使用。
- `m1` 是 `bytes_to_long()` 的结果，用 `long_to_bytes()` 还原。
- `m2` 是 `.hex()` 的结果，用 `bytes.fromhex()` 还原。
- `m3` 是 `b64encode()` 的结果，用 `b64decode()` 还原。

```python
from base64 import b64decode
from Crypto.Util.number import long_to_bytes

m0 = b"0xGame{73d7"
m1 = 60928972245886112747629873
m2 = "3165662d393339332d3034"
m3 = b"N2YwZTdjNGRlMX0="

flag = m0 + long_to_bytes(m1) + bytes.fromhex(m2) + b64decode(m3)
print(flag.decode())
```

运行结果为：

```text
0xGame{73d72f64-7656-11ef-9393-047f0e7c4de1}
```

## 方法总结

- 核心技巧：识别并逆转 bytes、整数、hex 与 Base64 四种常见表示。
- 识别信号：Python 输出中出现 `b'...'`、大整数、仅含十六进制字符的偶数长度字符串和带 `=` 填充的 Base64。
- 复用要点：表示层转换不是加密；恢复时注意 `bytes_to_long()` 使用大端字节序，并在最终拼接前统一为 `bytes`。
