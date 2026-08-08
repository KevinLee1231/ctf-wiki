# MiniLCTF 2021 - catch_cat（抓猫猫）

## 题目简述

两人轮流抓猫，第一次最多抓总数减一只，之后每次抓取数量不能超过上一手；抓走最后一只者获胜。程序先手，但题目保证它的第一步不是最优解。该题的决定性障碍是组合博弈，不属于现有密码、取证、隐写、Pwn 等技术方向，因此归入 `_unclassified`。

需要注意，欢迎文本写的是“每次必须大于 1”，实际校验却是 `number > 0`，所以抓 1 只合法。求解必须以源码中的检查为准。

## 解题过程

结论是：轮到某方行动时，如果剩余数量 $n$ 是 2 的幂，则该方处于必败态；否则可以取走

$$
\operatorname{lowbit}(n)=n\mathbin{\&}(-n)
$$

只猫，把局面交给对手。

证明可以按对手下一手 $a$ 与本手 $b=\operatorname{lowbit}(n)$ 的关系分类：

- 若 $a=b$，则 $n-2b$ 的最低有效位仍是 $b$，下一手仍可取 $b$；
- 若 $a<b$，因为 $b$ 是 2 的幂，减去由更低二进制位组成的 $a$ 后，新余数的 `lowbit` 必小于 $b$，也就不超过对方刚取的 $a$；
- 由于 $n$ 不是 2 的幂，始终有 $n-\operatorname{lowbit}(n)>\operatorname{lowbit}(n)$，所以对手不能在中途直接取完。

当 $n$ 是 2 的幂时，`lowbit(n) == n`，但规则禁止首手取完；任何合法动作都会把一个非 2 的幂局面交给对手，因此必败。

服务端的第一步故意不是最优解。收到它抓取后的剩余数量 `cats` 后，每轮发送 `lowbit(cats)`，再解析下一次剩余数量即可：

```python
import re
from pwn import remote

p = remote("127.0.0.1", 10000)

while True:
    line = p.recvline()
    print(line.decode(errors="replace"), end="")
    if b"The number of cats in XDU is" in line:
        cats = int(re.search(rb"(\d+)", line).group(1))
    if b"The number of cats you will catch is:" in line:
        take = cats & -cats
        p.sendline(str(take).encode())
    if b"You win!" in line:
        print(p.recvline().decode(errors="replace"), end="")
        break
```

公开参赛 WP 记录的远程 flag 为：

```text
miniL{c6d5bfc0-c92e-4d05-9d08-724667ee5900}
```

它来自当时的运行环境变量，不是源码中的固定常量。上述结论和证明也可与[出题人复盘](https://cdcq.github.io/2021/05/13/20210513a/)相互核对，但正文已经包含完整策略，无需依赖外链理解解法。

## 方法总结

这题最容易犯的错误是只凭样例总结“奇偶性”或照欢迎文本实现最小取 2。可靠做法是先读实际边界检查，再寻找状态的不变量。`lowbit` 把非 2 的幂局面逐步压向 2 的幂必败态，给出了短小、可证明、可自动化的必胜策略。
