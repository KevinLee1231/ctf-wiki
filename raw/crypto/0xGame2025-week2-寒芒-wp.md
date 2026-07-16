# 寒芒

## 题目简述

服务提供 AES-ECB 加密 oracle，实际明文形如 `user_input || secret`，整体再做 PKCS#7 填充。ECB 对相同 16 字节明文块产生相同密文块，因此可以用选择明文逐字节构造“候选块”，与包含未知 secret 的目标块比较。

## 解题过程

### 确认长度与模式

PKCS#7 总会填充至少 1 字节，每个填充值都等于填充长度。逐渐增加输入长度，记录密文第一次增加 16 字节时所需的输入量 $n$；若空输入密文长为 $L$，则 secret 长度为 $L-n$。

再提交两个以上完全相同的明文块。如果对应密文块也相同，即可确认 ECB 的独立分组特征。

### 逐字节恢复

恢复第 $i$ 个字节时，提交 $15-(i\bmod16)$ 个 `A`，把未知字节移动到当前目标块末尾。记录该密文块，然后枚举候选字节，将“对齐前缀 + 已恢复内容 + 候选”作为新输入；候选块密文与目标块相等时，该候选就是正确字节。

```python
BLOCK = 16

def recover_secret(oracle, secret_length):
    recovered = bytearray()
    for i in range(secret_length):
        pad_len = BLOCK - 1 - (i % BLOCK)
        prefix = b'A' * pad_len
        block_index = i // BLOCK
        target = oracle(prefix)[block_index * BLOCK:(block_index + 1) * BLOCK]

        for guess in range(256):
            probe = prefix + recovered + bytes([guess])
            candidate = oracle(probe)[block_index * BLOCK:(block_index + 1) * BLOCK]
            if candidate == target:
                recovered.append(guess)
                break
        else:
            raise ValueError(f'no candidate at offset {i}')
    return bytes(recovered)
```

`oracle()` 只需封装一次服务交互并返回原始密文字节。算法从前向后恢复 secret，不依赖猜测明文可读性。

## 方法总结

- 核心技巧：利用 ECB 的确定性分组映射执行 byte-at-a-time chosen-plaintext attack。
- 识别信号：攻击者输入与秘密拼接、固定 key 的 ECB oracle、可重复查询密文。
- 复用要点：先确定分组大小、secret 长度和前缀对齐；若服务存在未知随机前缀，还需先求前缀长度再调整块索引。
