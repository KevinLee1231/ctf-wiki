# 高等数学

## 题目简述

程序生成 120 道二元运算题，每 40 题为一组。真实运算符由 `tmp = rand() % 6` 决定，但输出给选手的运算符是 `sym[tmp + offset]`。`sym` 将 `+-x%^&` 连写两遍，因此该索引等价于对六种运算符循环平移。

每组第一题都会重新生成一次固定的 `offset`：

```c
int tmp = rand() % 6;
if (!flag1) {
    flag1++;
    srand(rand());
    offset = rand() % 6;
}
printf("%d %c %d = \n", Num1[idx], sym[tmp + offset], Num2[idx]);
scanf("%d", &res);
true = Trueval(Num1[idx], sym[tmp], Num2[idx]);
```

初始种子来自 `/dev/urandom`。远程无法直接复现随机序列，而程序也没有泄露足够状态；最简单的路线是反复连接，等待三组的 `offset` 都为 0。

## 解题过程

当某组 `offset = 0` 时，显示的运算符就是真实运算符，该组 40 题可全部正常计算。忽略极少数“两个不同运算恰好得到同一结果”的偶然情况，一次连接通过三组的概率为：

$$
\left(\frac{1}{6}\right)^3=\frac{1}{216}
$$

因此预期约需 216 次连接。脚本不使用 `eval`，而是显式实现六种运算；一旦收到 `You lose` 就关闭当前连接并重试：

```python
from pwn import *

HOST = args.HOST or "127.0.0.1"
PORT = int(args.PORT or 10000)


def calculate(left: int, operator: str, right: int) -> int:
    operations = {
        "+": lambda a, b: a + b,
        "-": lambda a, b: a - b,
        "x": lambda a, b: a * b,
        "%": lambda a, b: a % b,
        "^": lambda a, b: a ^ b,
        "&": lambda a, b: a & b,
    }
    return operations[operator](left, right)


attempt = 0
while True:
    attempt += 1
    io = remote(HOST, PORT)
    io.recvuntil(b"Can you finish this test?\n")

    failed = False
    for _ in range(120):
        expression = io.recvuntil(b" = \n", drop=True).decode()
        left, operator, right = expression.split()
        result = calculate(int(left), operator, int(right))
        io.sendline(str(result).encode())

        status = io.recvline()
        if b"You lose" in status:
            failed = True
            break

        if b"Good" not in status:
            raise RuntimeError(f"unexpected response: {status!r}")
        io.recvuntil(b"Next task\n")

    if failed:
        io.close()
        continue

    log.success(f"passed after {attempt} attempts")
    print(io.recvall(timeout=2).decode(errors="replace"))
    break
```

运行时通过参数提供题目实例：

```bash
python solve.py HOST=<主机> PORT=<端口>
```

## 方法总结

这道题的关键不是预测 `/dev/urandom`，而是识别“每 40 题共享一个六选一偏移”。只要偏移为 0，输出表达式就可信；三个独立分组同时满足的主概率为 $1/216$，适合自动重连。题解中的重试脚本应显式解析运算符，避免把远程输入直接交给 `eval`，同时不保留已经失效的比赛地址。
