# 觅码

## 题目简述

源码先把 UTF-8 编码后的 flag 等分为四段，再分别进行五次方、二进制文本、Base64 和十六进制编码。四段的切分发生在字节层，可能把一个中文字符的多字节编码拆开，因此不能对每段单独执行 UTF-8 解码。

## 解题过程

四段分别逆向：

1. `c1 = bytes_to_long(part1) ** 5`，对 `c1` 求精确五次整数根，再转回字节；
2. `c2` 是把每个字节的二进制表示直接拼接出的字符串，按二进制整数解析后转回字节；
3. `c3` 使用标准 Base64，直接解码；
4. `c4` 是十六进制文本，用 `bytes.fromhex()` 恢复。

脚本中的四个密文替换为题目输出。必须先拼接字节，再统一解码：

```python
from base64 import b64decode
from gmpy2 import iroot
from Crypto.Util.number import long_to_bytes

c1 = ...  # 题目给出的五次方整数
c2 = "..."  # 二进制字符串
c3 = b"..."  # Base64 字节串
c4 = "..."  # 十六进制字符串

root, exact = iroot(c1, 5)
assert exact

part1 = long_to_bytes(int(root))
part2 = long_to_bytes(int(c2, 2))
part3 = b64decode(c3)
part4 = bytes.fromhex(c4)

flag = (part1 + part2 + part3 + part4).decode("utf-8")
print(flag)
```

`exact` 为真说明 `c1` 确实是完整五次方，而不是需要模运算或近似求根的情形。

## 方法总结

本题考查“编码可逆、幂运算在无模数时可直接开根”以及文本与字节的边界。处理多字节字符时，应保持中间结果为 `bytes`，等完整字节流恢复后再按 UTF-8 解码，避免跨分段字符导致 `UnicodeDecodeError`。
