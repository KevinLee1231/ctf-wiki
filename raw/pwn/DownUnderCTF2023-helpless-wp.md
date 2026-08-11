# DownUnderCTF 2023 helpless Writeup

## 题目简述

SSH 用户的登录 shell 被错误设置成只执行 Python 的 `help()`。flag 位于 `/home/ductf/flag.txt`，但当前界面没有普通 shell 命令执行能力。目标是利用帮助系统调用的分页器读取任意文件。

## 解题过程

登录后，在 Python 帮助提示符中输入一个内容较长的模块名：

```text
help> os
```

帮助文本会交给 `less` 分页显示。`less` 支持 `:e` 命令打开另一个文件，因此在分页器中输入：

```text
:e/home/ductf/flag.txt
```

分页器随即显示目标文件：

```text
DUCTF{sometimes_less_is_more}
```

整个过程不需要退出到操作系统 shell，也不需要执行 Python 表达式；只使用了 `help()` 原本允许的模块查询和其下游分页器功能。

## 方法总结

受限交互环境的攻击面不仅是当前解释器，还包括它调用的编辑器、分页器和帮助查看器。`help()` 本身看似只读，但 `less` 的文件打开命令跨越了预期边界。审计 jail 时应检查完整子进程链和每个辅助工具的交互命令。
