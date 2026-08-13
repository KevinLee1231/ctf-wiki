# handson-000

## 题目简述

服务连续生成 10 道随机整数加、减或乘法题，并记录从程序启动起经过的时间。它被放在 Pwn 训练目录用于练习 pwntools 交互，但没有漏洞利用或其它安全机制，决定性任务只是脚本化协议交互，因此暂存 `_unclassified`。

## 解题过程

每轮输出题号和形如 `a + b =` 的表达式。解析三个 token 后按运算符计算，再把十进制结果发回：

```python
from operator import add, mul, sub
from pwn import process

ops = {"+": add, "-": sub, "*": mul}
io = process(["python3", "chall.py"])

for _ in range(10):
    io.recvuntil(b"/ 10")
    io.recvline()
    expression = io.recvuntil(b"=", drop=True).decode().strip()
    left, symbol, right = expression.split()
    answer = ops[symbol](int(left), int(right))
    io.sendline(str(answer).encode())

print(io.recvall().decode())
```

虽然程序会在总耗时超过 5 秒时打印 `Took too long!`，源码没有在该分支退出或判错，所以真正的成功条件只是十题全部回答正确。完成后得到：

```text
flag{m4th_g3niu5}
```

## 方法总结

处理交互式算术题时，应先固定提示边界，再解析结构化 token，避免直接对不可信文本使用 `eval`。同时必须阅读服务端判定代码：题面看似存在时间限制，但本题超时只产生提示，不影响最终成功状态。
