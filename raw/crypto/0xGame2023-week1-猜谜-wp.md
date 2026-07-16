# 猜谜

## 题目简述

密文先经过一个自定义的可打印编码，再由 7 字节循环密钥加密。自定义编码每次取 3 位作为下标，虽然字符表有 64 个字符，实际只会使用下标 0 到 7；解开表示层后，可利用已知 flag 前缀 `0xGame{` 恢复完整 7 字节密钥。

## 解题过程

内层加密满足：

$$
E_i=K_{i\bmod7}\oplus(P_i+i).
$$

前 7 个明文字节已知，且恰好覆盖一个完整密钥周期，所以：

$$
K_i=E_i\oplus(P_i+i),\qquad 0\le i<7.
$$

外层编码的逆过程是：去掉末尾 `=`，把每个字符在自定义表中的下标写成 3 位二进制，删掉编码时补入的 1 或 2 个零位，再每 8 位恢复一个字节。

```python
alphabet = "AP3IXYxn4DmwqOlT0Q/JbKFecN8isvE6gWrto+yf7M5d2pjBuk1Hh9aCRZGUVzLS"
ciphertext = b"IPxYIYPYXPAn3nXX3IXA3YIAPn3xAYnYnPIIPAYYIA3nxxInXAYnIPAIxnXYYYIXIIPAXn3XYXIYAA3AXnx="

def decode_outer(data: bytes) -> bytes:
    text = data.decode()
    padding = len(text) - len(text.rstrip("="))
    text = text.rstrip("=")
    bits = "".join(f"{alphabet.index(ch):03b}" for ch in text)
    if padding:
        bits = bits[:-padding]
    return bytes(int(bits[i:i + 8], 2) for i in range(0, len(bits), 8))

enc = decode_outer(ciphertext)
known = b"0xGame{"
key = bytes(enc[i] ^ (known[i] + i) for i in range(7))
flag = bytes((enc[i] ^ key[i % 7]) - i for i in range(len(enc)))
print(flag)
```

输出符合完整 flag 格式，即可同时验证外层解码、密钥恢复和字节运算均无偏移错误。

## 方法总结

已知明文攻击的关键是把加密式代数化，并检查已知片段是否覆盖完整密钥周期。本题外层编码只改变表示，不增加密码强度；正文给出了字符表、位宽、填充和逆变换，因此无需依赖外部 Base64 工具也能复现。
