# 高数大师

## 题目简述

服务端随机生成由 `sin(x)`、`cos(x)` 和 $x$ 到 $x^4$ 组成的表达式，并在末尾用 `(d)` 或 `(i)` 指定求导或不定积分。连续答对 300 题才会输出 flag，任意一题错误都会立即结束，因此需要自动解析题目并调用符号运算。

## 解题过程

阅读 `server.py` 可知，每一项的系数只会是 1 到 5，服务端使用 `sympy.diff` 或 `sympy.integrate` 生成标准答案，随后对双方结果执行 `sympy.simplify` 再比较。客户端使用同一套符号运算规则即可稳定通过，无需手写微积分规则。

下面的脚本从命令行接收比赛环境的地址，避免把已经失效的历史服务端点写死在题解中：

```python
from pwn import args, remote
import sympy

x = sympy.Symbol("x")


def receive_problem(io):
    while True:
        line = io.recvline().decode().strip()
        if line.endswith("(d)") or line.endswith("(i)"):
            expression, marker = line.rsplit(" ", 1)
            expression = expression.removeprefix(">").strip()
            return expression, marker[1]


def solve(expression, method):
    parsed = sympy.sympify(expression, locals={"x": x})
    if method == "d":
        return sympy.diff(parsed, x)
    if method == "i":
        return sympy.integrate(parsed, x)
    raise ValueError(f"unknown method: {method}")


if not args.HOST or not args.PORT:
    raise SystemExit("usage: python exp.py HOST=<host> PORT=<port>")

io = remote(args.HOST, int(args.PORT))
io.recvuntil(b"press enter to start")
io.sendline()

for _ in range(300):
    expression, method = receive_problem(io)
    answer = solve(expression, method)
    io.recvuntil(b"your answer > ")
    io.sendline(str(answer).encode())

print(io.recvall().decode())
```

运行方式：

```powershell
python ".\exp.py" HOST=<题目主机> PORT=<题目端口>
```

连续通过 300 轮后得到：

```text
0xGame{Y0u_Ar3_such_A_H1gh_M4th_M4s7er}
```

## 方法总结

本题属于 Misc 自动化交互题，决定性障碍是可靠解析协议并复现服务端的 SymPy 运算，而不是密码学。解析时应从行尾取 `(d)`、`(i)` 标记，并等待明确的答题提示后再发送结果，避免网络分包或多余空行导致轮次错位。
