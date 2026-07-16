# Test your pwntools

## 题目简述

服务端在 10 秒内连续生成 100 道加、减、乘法题。每题两个操作数均小于 100，全部答对后程序执行 `system("/bin/sh")`。人工输入来不及，目标是用 Pwntools 自动解析表达式、计算并回传答案。

## 解题过程

服务端每轮的格式固定为：

```text
====Round n====
a op b =
```

使用运算符映射比直接对网络数据调用 `eval()` 更安全，也能明确题目只允许三种操作。连接参数从 `HOST`、`PORT` 环境变量读取：

```python
import os
from operator import add, mul, sub
from pwn import *

context.log_level = "error"
HOST = os.environ["HOST"]
PORT = int(os.environ["PORT"])
io = remote(HOST, PORT)
operations = {"+": add, "-": sub, "*": mul}

for _ in range(100):
    io.recvuntil(b"====\n")
    expression = io.recvuntil(b" = ", drop=True).decode()
    left, symbol, right = expression.split()
    answer = operations[symbol](int(left), int(right))
    io.sendline(str(answer).encode())
    io.recvline()  # Correct!

io.recvuntil(b"Here's your shell!\n")
io.sendline(b"cat /flag")
print(io.recvline().decode().strip())
```

脚本关闭调试级日志，避免大量终端输出拖慢 100 轮交互。Pwntools 还能直接发送任意字节，因此后续二进制题中不受终端输入能力限制。仓库没有提交容器实际使用的 flag 文件，最终字符串应以脚本从当前实例读到的内容为准。

## 方法总结

- 核心技巧：使用 Pwntools 按协议同步接收、解析和发送 100 轮计算结果。
- 识别信号：交互格式重复、计算规则固定、服务端设置很短的 `alarm()`。
- 复用要点：每轮都要消费完提示和结果行，避免收发错位；在严格超时题中减少日志、正则和不必要等待。
