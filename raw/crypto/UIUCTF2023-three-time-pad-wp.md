# UIUCTF 2023 Three-Time Pad Writeup

## 题目简述

题目给出三份二进制密文 `c1`、`c2`、`c3`，并额外泄露第二条消息的明文 `p2`。三条消息均使用异或流加密，而且错误地重复使用了同一段 one-time pad。

若明文为 $P$、密钥流为 $K$、密文为 $C$，加密关系为：

$$
C=P\oplus K.
$$

## 解题过程

已知第二组明密文，因此可以直接恢复被复用的密钥流：

$$
K=C_2\oplus P_2.
$$

异或具有自反性，再将该密钥流分别与另外两份密文异或：

$$
P_1=C_1\oplus K,\quad P_3=C_3\oplus K.
$$

处理时必须以二进制模式读取文件，并使用 `zip` 按字节异或。题目各文件长度与所需密钥流范围相符，脚本如下：

```python
from pathlib import Path


def xor_bytes(left, right):
    return bytes(a ^ b for a, b in zip(left, right))


c1 = Path("c1").read_bytes()
c2 = Path("c2").read_bytes()
c3 = Path("c3").read_bytes()
p2 = Path("p2").read_bytes()

pad = xor_bytes(c2, p2)
p1 = xor_bytes(c1, pad)
p3 = xor_bytes(c3, pad)

print(p1.decode("ascii"))
print(p2.decode("ascii"))
print(p3.decode("ascii"))
```

实际输出为：

```text
before computers, one-time pads were sometimes
printed on flammable material so that spies could
uiuctf{burn_3ach_k3y_aft3r_us1ng_1t}
```

## 方法总结

一次一密的安全性依赖“密钥流真正随机、与消息等长且绝不复用”。一旦同一密钥流被复用，已知任意一组明密文就能恢复对应长度的密钥流，并解密其他消息；即使没有已知明文，也可通过 $C_1\oplus C_2=P_1\oplus P_2$ 消去密钥流，再结合语言统计分析。流密码和 CTR 模式同样必须避免 nonce 或密钥流复用。
