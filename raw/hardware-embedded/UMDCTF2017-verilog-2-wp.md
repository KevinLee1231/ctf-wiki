# Verilog 2

## 题目简述

题目给出两个 RSA 密文块和一个未完成的 Verilog 加密电路。源码没有实现 `TODO: modular exponentiation`，但在前 32 个时钟周期内通过大量 4 bit 切片赋值，逐步拼出了 RSA 模数 `n` 和私钥指数 `d`。因此目标不是补全硬件，而是静态求值这两组寄存器。

## 解题过程

逐个跟踪 `case(count)`。例如初始赋值：

```verilog
n[127:124] <= 1 + 2 + 3;
d[127:124] <= -2 + (4 << 1);
```

两者都得到高位 nibble `6`。后续赋值会引用已经确定的 nibble，再经过移位、异或、加减和按位运算写入其他位置。按非阻塞赋值语义逐周期求值，最终得到：

```text
n = 0x6a9d76542ea531fb10c1dc886c89bf43
d = 0x6457f6970a690fa1a721b05762e10471
```

题目已经给出两个十六进制密文。直接计算 $m=c^d\bmod n$：

```python
n = 0x6A9D76542EA531FB10C1DC886C89BF43
d = 0x6457F6970A690FA1A721B05762E10471
ciphertexts = [
    0x465EA069316D0F4D57F4152C3ABACE94,
    0xDBF62312BA954929E819D3503F1D46,
]

parts = []
for ciphertext in ciphertexts:
    message = pow(ciphertext, d, n)
    raw = message.to_bytes((message.bit_length() + 7) // 8, "big")
    parts.append(raw.decode())

print("_".join(parts))
```

两个明文块分别是：

```text
hardware
engineer
```

最终 flag：

```text
UMDCTF-{hardware_engineer}
```

其 SHA-256 与 README 中的摘要一致。

## 方法总结

硬件描述语言中的寄存器切片经常被用来隐藏常量。处理此类题目时要保留位宽、切片位置和非阻塞赋值的周期边界；得到 `n`、`d` 后，剩余部分就是普通 RSA 私钥运算。本题的模块名叫 `encrypt`，但真正泄露的是解密所需的私钥指数。
