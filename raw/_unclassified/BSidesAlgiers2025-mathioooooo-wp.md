# BSidesAlgiers2025 - mathIOOOOOO

## 题目简述

该题是交互数学题：服务每轮下发一个整数 `n`，要求在 1 秒内返回最小块数。
`chall.py` 直接给出参数生成与判定逻辑：

- `k = random.randint(10**2, 10**3)`
- `n = k * k`
- 标准答案是 `n + 2 * isqrt(n) - 3`
- 共 1000 轮，通过全部后返回 flag

因此题目的核心障碍是“识别出服务器输出的数列结构并构建稳定、低延迟的应答器”，不是暴力搜索或外部解密。

## 解题过程

### 关键观察

`chall.py` 中的正确答案函数是：

```python
correct_answer = n + 2 * int(math.isqrt(n)) - 3
```

服务约束 `n = k*k`（$k\in[100,1000]$）使得 `isqrt(n) = k`。代入可得闭式：

$$
ans = k^2 + 2k - 3
$$

由于题目要求的是每轮 1 秒内响应，直接按公式计算是正确且高效的闭环，不需要枚举、回溯或重型数学库。

### 求解步骤

1. 用正则从服务输出行抽取 `n`。
2. 使用整数平方根 `isqrt(n)` 防止浮点误差。
3. 计算 `n + 2*isqrt(n) - 3` 并立即提交。
4. 连续完成 1000 轮后等待 flag 行。

整理成不绑定失效比赛端点的完整客户端如下：

```python
import argparse
import math
import re

from pwn import process, remote

parser = argparse.ArgumentParser()
parser.add_argument("--local", action="store_true")
parser.add_argument("--host")
parser.add_argument("--port", type=int)
args = parser.parse_args()

if args.local:
    io = process(["python3", "challenge/chall.py"])
else:
    if args.host is None or args.port is None:
        parser.error("远程模式需要 --host 和 --port")
    io = remote(args.host, args.port)

n_regex = re.compile(rb"n\s*=\s*(\d+)")

while True:
    data = io.recvline(timeout=2)
    if not data:
        break
    print(data.decode(errors="replace").rstrip())
    if b"shellmates{" in data.lower():
        break
    match = n_regex.search(data)
    if match:
        n = int(match.group(1))
        answer = n + 2 * math.isqrt(n) - 3
        io.sendline(str(answer).encode())
```

本地复现可执行 `python3 solve.py --local`；远程复现则显式传入当前实例的 `--host` 与 `--port`。这样保留了官方 solver 的交互逻辑，但不把已经失效的占位地址写成可用服务。

### 验证

闭环条件是：只要每轮按公式提交正确答案，服务不触发 `EOF`、超时或错误分支即可继续到下一轮。完成 1000 轮后，仓库内 `challenge/flag.txt` 对应的结果为：

`shellmates{y00uuuuu_ar4_an_LLM_exp4rt_gu3ss3r!}`

这个检查标准完全由源码给定，脚本可复用，不依赖额外外部数据。本题的决定性障碍只是闭式计算与高频交互，不涉及密码体制，因此归入 `_unclassified`，而不是仅因题面出现数学表达式就误归为 `crypto`。

## 方法总结

- 关键技巧：优先读源码，确认 `correct_answer`，再将其转成 $O(1)$ 公式计算，比盲猜模型或搜索快得多。
- 识别信号：服务端明示了 `isqrt` 或有明显单调结构时，优先判断为“数学直接计算”类题，而非算法搜索。
- 复用要点：在高频交互题里，核心不是“更聪明的算法”，而是“更快的通信与解析循环”：正则抽取、整数运算、无状态快速提交、超时防抖。
