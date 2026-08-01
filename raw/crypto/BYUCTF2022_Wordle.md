# BYUCTF 2022 - Wordle

## 题目简述

服务实现标准五字母 Wordle，但猜测接口不返回黑、黄、绿方块，只返回五个反馈符号连接后计算出的 MD5。目标是在最多六次猜测内找出单词，再调用结束接口领取 flag。

## 解题过程

每个位置只有三种状态，五个位置总共只有 `$3^5=243$` 种反馈。MD5 在这里不是需要碰撞的密码学难题，只是一个很小字典的编码。预先枚举全部状态即可建立反查表：

```python
import hashlib
from itertools import product

symbols = ['⬛', '🟨', '🟩']
lookup = {}
for state in product(symbols, repeat=5):
    s = ''.join(state)
    lookup[hashlib.md5(s.encode()).hexdigest()] = s
```

每次向 `/api/v1/guess/` 提交合法单词后，用返回的 `hashString` 查回真实反馈，再按普通 Wordle 规则缩小候选词。解出答案后，最后一次猜测必须等于服务端答案，再携带同一 `id` 与 `key` 调用 `/api/v1/finish_game/`。

成功响应为：

```text
Nice Job! byuctf{b@c0n_grease}
```

## 方法总结

哈希只隐藏了 243 种低熵状态，无法提供实际保密性。先界定输入空间，再决定是碰撞、穷举还是字典反查；本题穷举全部反馈比攻击 MD5 本身简单得多。
