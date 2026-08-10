# 奇妙小游戏

## 题目简述

服务连续给出 5 关由随机字符绘制的“鬼脚图”（Amidakuji），要求在很短时间内回答底部编号沿横线向上追踪后对应的顶部入口。决定性障碍是恢复文本图中的状态转移并自动交互，而不是利用内存漏洞，因此本文归入 Reverse。

## 解题过程

每条竖线的中心列相隔 5 个字符。设当前位置为 `position`，中心下标就是 `5 * position`。从图的最底行向上遍历：

- 若中心右侧字符不是空格，说明该层有向右的横线，执行 `position += 1`；
- 否则，若当前不是最左列且中心左侧字符不是空格，说明横线通向左侧，执行 `position -= 1`；
- 两侧都是空格时保持原编号。

题目会随机替换绘图用的方括号和横线字符，所以不能匹配某个固定符号，只判断“是否为空格”更稳定。核心求解函数为：

```python
def trace_ladder(rows: list[str], position: int) -> int:
    for row in reversed(rows):
        center = 5 * position

        right_connected = center + 1 < len(row) and row[center + 1] != " "
        left_connected = center > 0 and row[center - 1] != " "

        if right_connected:
            position += 1
        elif left_connected:
            position -= 1

    return position
```

将其接入服务，并补全官方 PDF 中缺失的导入和未定义变量，可得到如下交互骨架：

```python
#!/usr/bin/env python3
import hashlib
import string

from pwn import *
from pwnlib.util.iters import mbruteforce

HOST = "challenge.host"
PORT = 0

def trace_ladder(rows: list[str], position: int) -> int:
    for row in reversed(rows):
        center = 5 * position
        if center + 1 < len(row) and row[center + 1] != " ":
            position += 1
        elif center > 0 and row[center - 1] != " ":
            position -= 1
    return position

def solve_pow(io) -> None:
    io.recvuntil(b"sha256(????) == ")
    target = io.recvn(64).decode()
    proof = mbruteforce(
        lambda value: hashlib.sha256(value.encode()).hexdigest() == target,
        string.digits + string.ascii_letters,
        4,
        method="fixed",
    )
    io.sendlineafter(b"input your ????> ", proof.encode())

def solve_level(io) -> int:
    io.recvuntil(b"level:")
    level = int(io.recvline().strip())

    # 跳过图形前的标题行；若实际实例格式不同，应据输出调整这里。
    io.recvline()
    rows = []
    while True:
        row = io.recvline(keepends=False).decode()
        if len(row) > 1 and row[1] == "-":
            break
        rows.append(row + " ")

    io.recvuntil(b"is ")
    bottom = int(io.recvline().strip())
    answer = trace_ladder(rows, bottom)
    io.sendline(str(answer).encode())
    return level

io = remote(HOST, PORT)
solve_pow(io)
io.sendline(b"start")

while solve_level(io) < 5:
    pass

io.interactive()
```

服务在第 5 关通过后返回 flag。官方实例已关闭，文档和公开参赛记录都没有保存最终 flag 字符串，因此这里不伪造结果。公开题解验证了 5 关、5 字符列间距以及从下向上追踪的协议细节：[HGAME2022 Misc Writeup](https://www.cnblogs.com/zysgmzb/p/15882495.html)。

## 方法总结

这类限时文本游戏应先把画面抽象成状态机，再考虑网络自动化。随机替换绘图字符时，“非空即连边”比枚举可能符号更可靠；追踪方向必须从题目给出的端点反向决定。`pwntools` 只负责收发，真正决定解法的是对鬼脚图拓扑的正确建模。
