# Shaker

## 题目简述

服务把 64 字节 flag 作为状态，生成固定的 64 字节掩码 $x$ 和一个位置置换 $P$。一次 `shake` 执行先异或 $x$、再按 $P$ 重排；`open` 先异或 $x$ 并返回状态，随后重新随机打乱 $P$，再执行一次 `shake`。flag 的 MD5 已公开，可用于判定某次输出是否恰好回到原文。

## 解题过程

固定一次 `open` 后的新置换，后续 `shake` 是有限状态空间上的仿射置换：

$$
T(s)=P(s\oplus x).
$$

第一次 `open` 返回的并不是所需明文，但其内部 `reset` 把状态推进到新置换下的起点。官方解法随后执行 119 次 `shake`，再第二次 `open`；连同 `reset` 内的一次推进，等价于考察 120 步周期。

随机置换的循环长度组合有时会让相应仿射变换的周期整除 120，此时第二次输出恰好恢复 flag。不是每个连接都会命中，因此用公开 MD5 筛选并重连：

```python
import hashlib
from pwn import remote

expected = "4839d730994228d53f64f0dca6488f8d"

for attempt in range(1000):
    io = remote("target.example", 33302)

    io.sendlineafter(b"> ", b"2")
    io.recvuntil(b"Result: ")
    io.recvline()

    for _ in range(119):
        io.sendlineafter(b"> ", b"1")

    io.sendlineafter(b"> ", b"2")
    io.recvuntil(b"Result: ")
    candidate = bytes.fromhex(io.recvline().strip().decode())

    if hashlib.md5(candidate).hexdigest() == expected:
        print(candidate.decode())
        break
    io.close()
```

本地按相同状态转移模拟随机会话，可以在有界重试内命中，候选为：

```text
grey{kinda_long_flag_but_whatever_65k2n427c61ww064ac3vhzigae2qg}
```

## 方法总结

- 核心技巧：把“异或后置换”视为有限仿射置换，利用随机置换偶尔出现的短周期让状态回到明文。
- 识别信号：状态长度固定、变换可逆、密钥和置换在多轮间固定，并且提供明文哈希时，应检查变换阶和周期碰撞。
- 复用要点：120 轮包含 `open` 触发的内部 `reset` 推进一步，因此显式菜单只需再 shake 119 次；必须用哈希验证，不能把任意可打印输出误当 flag。
