# Reorder

## 题目简述

服务生成一个固定的 16 元素置换 `PBOX`，把输入补齐为 16 字节分组后，对每组执行 `output[j] = input[PBOX[j]]`。用户可以提交最多 10 次明文并观察密文，随后服务给出使用同一置换处理的 flag。

## 解题过程

置换只改变位置，不改变字节值。提交一个 16 字节互不相同的已知分组即可一次恢复整个 `PBOX`：

```text
0123456789abcdef
```

如果对应输出记为 `known_cipher`，那么输出位置 `j` 上的字符来自输入位置 `known.index(known_cipher[j])`，这正是 `PBOX[j]`。解密时执行逆置换，把密文字节放回原位置：

```python
BLOCK_SIZE = 16
known = b"0123456789abcdef"


def recover_pbox(known_cipher: bytes) -> list[int]:
    assert len(known_cipher) == BLOCK_SIZE
    return [known.index(byte) for byte in known_cipher]


def decrypt_block(cipher_block: bytes, pbox: list[int]) -> bytes:
    assert len(cipher_block) == BLOCK_SIZE
    plain = bytearray(BLOCK_SIZE)
    for output_pos, input_pos in enumerate(pbox):
        plain[input_pos] = cipher_block[output_pos]
    return bytes(plain)


def decrypt(ciphertext: bytes, pbox: list[int]) -> bytes:
    assert len(ciphertext) % BLOCK_SIZE == 0
    return b"".join(
        decrypt_block(ciphertext[i:i + BLOCK_SIZE], pbox)
        for i in range(0, len(ciphertext), BLOCK_SIZE)
    ).rstrip(b" ")


# 将服务对 known 的返回和最后输出的加密 flag 填入这里。
known_cipher = bytes.fromhex("<known-cipher-hex>")
encrypted_flag = bytes.fromhex("<encrypted-flag-hex>")

pbox = recover_pbox(known_cipher)
print(decrypt(encrypted_flag, pbox).decode())
```

这里必须从原始字节或无损的十六进制表示读取结果，不能让终端转义破坏非打印字节。官方 PDF 只展示了算法而未记录一次具体会话；公开的[参赛者复盘](https://asjet.dev/2020/02/week1/)记录的明文为：

```text
hgame{jU$t+5ImpL3_PeRmuTATi0n!!}
```

## 方法总结

- 核心技巧：对纯位置置换选择“每个位置的字节都唯一”的明文，一次查询即可确定完整映射。
- 识别信号：每个 16 字节分组只是按同一索引数组重排，且服务提供已知明文查询。
- 复用要点：恢复的是加密方向 `output[j] = input[PBOX[j]]`；解密必须把 `output[j]` 写回 `input[PBOX[j]]`，不要再次按同方向置换。
