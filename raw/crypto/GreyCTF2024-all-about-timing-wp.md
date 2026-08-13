# GreyCTF2024 All About Timing WP

## 题目简述

服务端执行 `random.seed(int(time.time()))`，随后用 Python 的 `random.randint` 生成一个 16 位十进制数。种子只有“当前 Unix 秒”这一项，而客户端和服务端通常处在同一秒内，因此随机数可以被完整预测。

## 解题过程

生成逻辑的关键部分是：

```python
random.seed(int(time.time()))
n = random.randint(10**15, 10**16 - 1)
```

Python 的 Mersenne Twister 是确定性伪随机数生成器。只要种子相同，第一次 `randint` 的输出也相同。连接后立即用本地当前秒播种，并调用一次相同的函数：

```python
from pwn import *
import random
import time

io = remote(args.HOST, int(args.PORT))
random.seed(int(time.time()))
guess = random.randint(10**15, 10**16 - 1)
io.sendlineafter(b"Your guess:", str(guess).encode())
io.interactive()
```

若连接恰好跨过秒边界，重新连接再试即可；也可以为当前时间前后各一秒分别建立连接。成功预测后得到：

```text
grey{t1m3_i5_a_s0c1al_coNstRucT}
```

## 方法总结

时间戳不是秘密，也不是高熵种子。看到 `int(time.time())` 与可知的 PRNG 调用顺序时，应在相同运行时中复刻“播种—首次取值”过程；真正需要处理的只有网络延迟造成的一秒边界误差。
