# RNG GAME

## 题目简述

Python random 种子碰撞题。revenge 版本预期分析 CPython `random` 底层 C 实现，在 seed 扩展填充状态数组阶段构造碰撞种子；早期版本还存在负数限制缺失导致的非预期。

## 解题过程

最早的题忘了限制负数，造成了非预期。revenge 版本预期是分析 Python `random` 底层 C 实现：seed 扩展阶段会通过特殊函数填充状态数组，最终可以构造种子碰撞。官方 WP 没有展开推导，直接给出 exploit。

```python
from pwn import *
# context(log_level = 'debug')
addr = "challenge.host:PORT".split(':')
io = remote(addr[0], int(addr[1]), ssl=False)
io.recvuntil("Here is my seed: ")
seed = int(io.recvline().strip())
tmp = []
while seed:
    tmp.append(seed & 0xffffffff)
    seed >>= 32
for i in range(4):
    tmp.append(tmp[i] - 4)
seed2 = sum([tmp[i] << (32 * i) for i in range(8)])
io.recvuntil("Give me your seed: ")
io.sendline(str(seed2))
io.interactive()
```

## 方法总结

- 核心技巧：构造 Python random seed 扩展阶段的状态碰撞。
- 识别信号：服务端给出 seed 并要求提交另一个 seed 时，应检查 seed 到 MT 状态数组的扩展过程是否可碰撞。
- 复用要点：按 32 位分块处理大整数 seed，构造补偿分块使扩展后的状态满足同一随机序列。
