# X Marked the Spot

## 题目简述

附件把 48 字节 Flag 与循环使用的 8 字节密钥逐字节异或，并只给出密文 `ct`。重复密钥 XOR 本身没有隐藏长度信息，而标准 Flag 前缀和末尾右花括号提供了恰好覆盖全部 8 个密钥位置的已知明文。

## 解题过程

加密关系为 $C_i=P_i\oplus K_{i\bmod 8}$。由 XOR 的自反性可得 $K_{i\bmod8}=C_i\oplus P_i$。已知前缀 `uiuctf{` 长 7 字节，因此能直接恢复 `key[0:7]`。

剩余的 `key[7]` 不能由开头第 8 个未知 Flag 字节得到，但 Flag 总长为 48，满足 $48\equiv0\pmod8$。最后一个字符 `}` 位于索引 47，而 $47\bmod8=7$，正好泄漏最后一个密钥字节。完整恢复代码如下：

```python
from itertools import cycle

with open("ct", "rb") as file:
    ciphertext = file.read()

key = bytearray(c ^ p for c, p in zip(ciphertext, b"uiuctf{"))
key.append(ciphertext[-1] ^ ord("}"))

plaintext = bytes(c ^ k for c, k in zip(ciphertext, cycle(key)))
print(bytes(key))
print(plaintext.decode())
```

得到密钥和 Flag：

```text
hdiqbfjq
uiuctf{n0t_ju5t_th3_st4rt_but_4l50_th3_3nd!!!!!}
```

## 方法总结

- 重复密钥 XOR 不是一次一密；同一密钥位置会周期性复用，任意已知明文都能恢复对应位置的密钥。
- 已知前缀不一定覆盖整个周期。本题还利用固定末尾 `}` 和总长度模周期的关系补齐了第 8 个字节。
- 恢复密钥后应对整段密文重新异或，并检查前缀、右花括号及可打印性，以排除下标或周期判断错误。
