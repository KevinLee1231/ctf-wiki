# ecb-a-tron-9000

## 题目简述

服务把攻击者可控明文与固定秘密串联为 `input || SECRET`，用空格补齐到 16 字节，再使用同一随机密钥进行 AES-ECB 加密。接口同时返回完整密文并支持批量查询。由于 ECB 对相同明文块产生相同密文块，可以逐字节建立 codebook，恢复只含大写字母的秘密。

## 解题过程

AES 块大小为 16 字节。要恢复秘密的第一个字符，先提交 15 个已知字节，使第一块恰好为：

```text
AAAAAAAAAAAAAAA?
```

其中 `?` 是 `SECRET[0]`。记录该块密文，再分别查询：

```text
AAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAB
...
AAAAAAAAAAAAAAAZ
```

当某个候选块的密文与目标块相同，ECB 的确定性说明候选末字节就是秘密首字节。本题匹配到 `D`。

恢复后续字节时，每轮减少一个前缀填充字节，并把已知秘密拼进候选块。设已恢复前缀为 `known`，第 $i$ 轮使用：

```python
pad = b"A" * (15 - (i % 16))
target_block = (i // 16)

# oracle(x) 返回 AES-ECB(input=x || SECRET) 的密文
target = oracle(pad)[16 * target_block:16 * (target_block + 1)]

for guess in b"ABCDEFGHIJKLMNOPQRSTUVWXYZ":
    probe = pad + known + bytes([guess])
    block = oracle(probe)[16 * target_block:16 * (target_block + 1)]
    if block == target:
        known += bytes([guess])
        break
```

逐字节得到秘密：

```text
DONTUSEECBPLEASE
```

前端说明要求把秘密包裹为 flag，因此结果是：

```text
DUCTF{DONTUSEECBPLEASE}
```

## 方法总结

- 核心技巧：对 `attacker_input || secret` 的 ECB oracle 做 byte-at-a-time codebook attack。
- 识别信号：同一输入总得到同一密文、块大小固定为 16 字节、攻击者能控制秘密前方的明文，并能观察各密文块。
- 复用要点：每轮都要让未知字节落在目标块末尾；跨块后同步调整块索引。随机生成一次会话密钥并不能阻止攻击，问题在于 ECB 的独立确定性分块及可查询 oracle。
