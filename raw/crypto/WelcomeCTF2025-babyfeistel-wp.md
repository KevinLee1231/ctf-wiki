# Babyfeistel

## 题目简述

服务维护一个 32 字节内部状态。每轮把状态拆成两个 16 字节半块 $L,R$，计算：

$$
(L,R)\longmapsto(R,\operatorname{MD5}(L))
$$

加密消息时先按 32 字节 PKCS#7 填充，为每个明文块推进一次状态，把产生的状态块打乱后拼成一次性密钥，再与明文异或。服务同时提供任意明文加密和 flag 加密；固定实例中的状态会在多次请求之间继续推进。

## 解题过程

先提交 32 字节全零明文。全零与密钥流异或后仍是密钥流，因此响应的前 32 字节直接泄露一次完整轮状态。设泄露值为 $S_1$，从它开始按公开轮函数继续计算，就能预测后续状态：

```python
import hashlib


def next_state(state):
    left, right = state[:16], state[16:]
    return right + hashlib.md5(left).digest()
```

已知明文请求本身因填充会消耗两个 32 字节块，所以随后请求 `FLAG` 时，flag 使用的候选状态位于泄露状态之后。官方 solver 预先计算 10 个状态，再对 flag 密文的每个 32 字节块尝试所有候选：

```python
import string


states = []
state = leaked_state
for _ in range(10):
    state = next_state(state)
    states.append(state)

flag = bytearray()
for offset in range(0, len(encrypted_flag), 32):
    chunk = encrypted_flag[offset:offset + 32]
    for candidate in states:
        plain = bytes(a ^ b for a, b in zip(chunk, candidate))
        if all(chr(value) in string.printable for value in plain):
            flag.extend(plain)
            break
```

之所以要枚举候选，是因为服务对状态块执行了 `random.shuffle`，密文块顺序与状态生成顺序不再一致。可打印性和 `grey{...}` 格式用于识别正确排列，最后去掉 PKCS#7 填充即可得到：

```text
grey{feistel_cipher_skill_issue_0uCVXt7TtdfhwwaXpN6OGRkSuEWg5QMW7NNyTtlZemKD22DB8IEhZQAZ5SM0ziLBsJaTqhoXbzXBHcIIgjc2urmHe2zadH0qEOh9}
```

## 方法总结

- 核心技巧：用全零已知明文直接泄露异或密钥流，再按确定性状态转移预测后续密钥块。
- 识别信号：同一有状态实例同时提供任意明文与秘密明文加密，且加密主体只是明文与内部状态异或。
- 复用要点：必须把填充额外消耗的轮数和状态块乱序计入；候选明文应同时通过可打印性、flag 格式和填充验证。
