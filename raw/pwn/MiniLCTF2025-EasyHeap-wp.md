# EasyHeap

## 题目简述

程序提供按索引 add/edit/show/free 的堆菜单，释放后没有清空记录，直接形成 UAF。目标使用带 safe-linking 与 tcache key 检查的新版 glibc，并安装 seccomp：`open`、`openat` 被禁，但 `openat2`、`readv`、`writev` 与 `mprotect` 仍可组成替代链。官方解法通过重复清除 tcache key 释放同一 chunk，泄露 heap/libc，tcache poisoning 改写 `_IO_list_all`，再用 House of Illusion 取得栈地址和栈写，最终 `mprotect` 堆页并执行 shellcode。

仓库没有本题源码、ELF、libc 和 `SomeofHouse` 辅助库，所以无法确认 RELRO、canary、NX、PIE 或完整 seccomp BPF。本文中的 `0x203b20`、ROP gadget 和 FILE 字段偏移都来自官方脚本，只对其配套 libc 有效。

## 解题过程

### 为什么不能直接 `execve("/bin/sh")`

seccomp 表面上没有禁用 `execve`，但动态链接的 `/bin/sh` 启动后需要打开 `/etc/ld.so.cache` 和 libc，动态加载器会调用被禁用的 `openat`，进程随即因 bad syscall 终止。因此“允许 execve”不等于动态程序可以正常启动。

最终 shellcode改用：

```text
openat2(0, "/flag", &zero_open_how, 0x18)
readv(flag_fd, &iov, 1)
writev(1, &iov, 1)
```

绝对路径 `/flag` 不依赖 `dirfd`，清零的 `open_how` 表示只读打开。`readv/writev` 绕过了只针对常规文件调用的限制。

### 通过 UAF 重复释放同一 tcache chunk

先分配两个 `0x100` chunk 和一个隔断 chunk：

```python
add(0, 0x100, b"shellcode holder")
add(1, 0x100, b"tcache target")
add(31, 0x100, b"gap")
```

释放索引 1 后，索引仍保留旧指针。再分配索引 2 会复用同一地址，于是索引 1、2 成为别名；再次 `free(1)` 后，`show(2)` 仍能读取已释放 chunk。单元素 tcache 链的 mangled `fd` 等于 chunk 地址右移 12 位，因此：

$$
\text{heap base}=\text{leaked fd}\ll 12
$$

新版 glibc 的 double-free 检测还检查用户区第二个八字节的 tcache key。利用 `edit(2, b"a"*0x10)` 覆盖 `fd` 和 key，再通过陈旧索引 `free(1)`，即可反复释放同一物理 chunk。重复七轮填满对应 tcache bin，下一次释放进入 unsorted bin；`show(2)` 读到 `main_arena` 指针。官方 libc 的基址计算为：

```python
libc.address = leak - 0x203b20
```

### tcache poisoning 指向 `_IO_list_all`

safe-linking 编码为：

$$
\operatorname{PROTECT\_PTR}(pos,ptr)=(pos\gg 12)\oplus ptr
$$

官方布局中，受控 chunk 位于 `heap_base+0x590`。把其 tcache `fd` 改为 `_IO_list_all` 的 mangled 地址，再进行两次同尺寸分配，第二次分配便落到 `_IO_list_all`：

```python
chunk_addr = heap_base + 0x590
mangled = (chunk_addr >> 12) ^ libc.sym["_IO_list_all"]
edit(2, p64(mangled))
add(3, 0x100, b"consume poisoned chunk")
add(4, 0x100, p64(chunk_addr))  # _IO_list_all = chunk_addr
```

### House of Illusion 取得栈写

官方脚本在 `chunk_addr+0x400` 构造 House of Illusion 的伪 `_IO_FILE`，并把 `_IO_list_all` 链到该对象。选择菜单 5 触发退出/FILE 清理路径后，伪 FILE 的受控字段把一次内部 IO 转换为向攻击者指定区域的 `read`，辅助函数 `bomb_raw` 再泄露栈地址并把后续输入落到返回栈。

这一步依赖缺失的 `SomeofHouse.HouseOfSome` 实现。现有证据能确认伪 FILE 模板地址、`_IO_list_all` 改写和触发菜单，但无法仅从总 WP 恢复所有 FILE 字段，不能伪造一个所谓“独立可运行”的 House of Illusion 模板。

### 栈上 ROP 与沙箱绕过 shellcode

最终 ROP 把堆首页改成 RWX：

```text
rdi = heap_base
rsi = 0x1000
rdx = 7
call mprotect
jump heap_base + 0x480
```

`heap_base+0x480` 处预先写入 `openat2 → readv → writev` shellcode。至此利用链为：

```text
UAF 别名
  → 清 tcache key 重复 free
  → heap leak + unsorted-bin libc leak
  → safe-linking tcache poisoning
  → _IO_list_all / House of Illusion
  → stack leak + stack write
  → mprotect heap RWX
  → openat2/readv/writev 读取 flag
```

官方材料没有保存最终 flag 或运行输出，因此本文验证到的是脚本中完整的数据流和偏移关系，而非当前附件上的动态复现。

## 方法总结

- 核心技巧：用 UAF 别名清除 tcache key，反复释放同一 chunk，逐步获得 heap/libc 泄露、任意分配和 FILE 链控制。
- 识别信号：释放后索引未清空、edit 可覆盖 freed chunk 前 16 字节，说明 safe-linking 与 tcache key 都可能被同一个 UAF 原语绕过。
- 复用要点：seccomp 分析必须跟踪目标程序启动后的二次 syscall；`execve` 本身获准并不保证动态 shell 可运行。所有 safe-linking、FILE 和 gadget 偏移必须与目标 glibc 严格匹配。
