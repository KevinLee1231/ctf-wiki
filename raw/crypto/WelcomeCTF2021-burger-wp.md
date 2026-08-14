# Burger

## 题目简述

WelcomeCTF2021 的 Burger 提供一个分块哈希服务。服务把用户输入与 56 字节 flag 直接拼接，补零到 16 字节边界，再分别计算每块的 SHA-512，并只返回每块摘要的前 16 字节。目标是在最多 100 次查询内恢复 flag。

## 解题过程

核心代码等价于：

```python
text = user_input + FLAG
text += b"\0" * (16 - len(text) % 16)

for offset in range(0, len(text), 16):
    block = text[offset:offset + 16]
    result += hashlib.sha512(block).hexdigest()[:32]
```

每个块独立哈希，相同的 16 字节明文块必然产生相同结果。这和 ECB 字节逐位恢复的思路相同：调整可控前缀长度，使当前未知 flag 字节与已经恢复的后缀、确定的零填充共同落入一个块，然后在本地枚举该未知字节。

flag 长度为 56。第一次发送 9 字节输入时，`input || flag` 长度为 65，最后一块只包含 flag 最后一个字节与 15 个零。枚举 $0\ldots255$，本地计算

```python
def block_hash(block):
    return hashlib.sha512(block).hexdigest()[:32]

candidates = {
    block_hash(bytes([value]) + b"\0" * 15): value
    for value in range(256)
}
```

将远端最后一块摘要与字典匹配，就能恢复最后一个字节。随后把输入增加到 10 字节，目标块变为“倒数第二个字节、已知的最后一个字节、14 个零”，继续枚举倒数第二个字节。每恢复 16 个字节后，目标会移到前一个摘要块，因此官方脚本按

```python
block_index = -1 - (input_length - 8) // 16
```

选择对应块。重复该过程即可从右向左恢复完整 flag。外部哈希服务并不参与解题，所需的候选摘要都能用 Python 标准库本地计算。

最终得到：

```text
greyhats{B3lanJa_m3_Burg3R_1f_y0u_3njoyed_7he_Ch@ll3n93}
```

## 方法总结

哈希函数本身没有被逆向；漏洞来自“固定分块、逐块独立哈希、攻击者可控制 flag 前缀”这三个条件。通过对齐块边界，每次只保留一个未知字节，256 次本地枚举就足以确定它。分析这类 oracle 时应先画出每次查询后的真实分块，而不是尝试直接碰撞 SHA-512。
