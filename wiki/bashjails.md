---
type: family
tags: [misc, family, bash, jail, restricted-shell, shell-escape]
skills: [ctf-misc, ctf-pentest]
raw:
  - ../raw/misc/bashjails.md
updated: 2026-06-12
---

# Bash Jails & Restricted Shells

## 作用边界

本页是 bash/rbash/受限 shell 的 family 页，负责把字符集限制、eval 上下文、变量构造、文件读取、网络外带、post-shell 枚举和 rbash 环境变量滥用分流到可复现的逃逸路线。

它不处理 Python 对象模型类 jail，也不处理纯 Web 命令注入。判断标准很简单：如果限制发生在 shell 语法、内建命令、文件描述符、环境变量或 rbash 行为上，先看本页；如果限制发生在 Python namespace 或字节码层，转 [pyjails.md](pyjails.md)。

## 共同识别信号

- 远程服务把输入送进 `bash`、`sh`、`rbash`、`eval`、`read -e`、`printf`、`echo` 或受限命令白名单。
- 可用字符极少，例如只剩 `$`、`#`、反斜杠、数字、换行、引号、`=` 或环境变量切片。
- 输出被关闭、截断、重定向，或只能通过 `/dev/tcp`、fd 0、stderr、history、verbose mode 取回。
- escape 后还需要枚举内部服务、flag 进程、网络端口或本机提权线索。

## 最小证据

- 确认实际 shell、调用方式和 eval 层数：`bash -c`、脚本、交互 shell、`sh`、`rbash` 或 wrapper。
- 枚举允许字符、被过滤字符、可用内建、变量、文件描述符和输出通道。
- 能构造一个最小可见副作用：打印数字、读取环境变量、列目录、连回 socket 或写入文件。
- 明确 escape 目标：读文件、执行命令、外带、进入交互 shell、突破 rbash 或 post-shell 枚举。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| 不知道输入如何被执行 | 先用 harmless 字符和错误信息识别 eval 层数、引号环境、命令替换和 shell 类型 | [misc-tooling.md](misc-tooling.md) |
| 只能用 `$`、`#`、反斜杠或少量字符 | 先用 `$#`、`${##}`、`$$`、变量切片、ANSI-C quoting 或 PID 数字构造字节 | [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md) |
| 允许环境变量、`PATH`、`BASH_ENV`、`HISTFILE`、`LD_PRELOAD` | 先判断变量是否被 rbash/wrapper 接受，再用历史文件、preload 或启动文件侧写 | [source-backdoors-and-restricted-shell-tricks.md](source-backdoors-and-restricted-shell-tricks.md), [linux-privesc.md](linux-privesc.md) |
| stdout 关闭、只有 fd 0 或 CR 截断 | 先枚举 fd，再用 `1>&0`、stderr、`\r` 显示或 `/dev/tcp` 外带恢复输出 | [interactive-containers-jails-and-solvers.md](interactive-containers-jails-and-solvers.md) |
| 只能 echo/printf 分层构造 payload | 先生成中间脚本或 octal 字节串，再逐层解锁更多字符 | [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md) |
| 已拿到低权限 shell | 继续枚举进程、端口、计划任务、SUID、sudo、服务配置和内部 flag 服务 | [linux-privesc.md](linux-privesc.md), [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md) |
| 其实是 Python、Lua、Ruby 或 C jail | 不在 shell 语法里硬凑，转对应语言或运行时页面 | [pyjails.md](pyjails.md), [interactive-containers-jails-and-solvers.md](interactive-containers-jails-and-solvers.md) |

## 合并与拆分结论

本页应保留为 family。bash jail 的关键 pivot 是 eval 上下文、字符集、输出通道和 post-shell 目标；这些判断与 Python jail 不同，也不能并入通用 Misc 首轮页。当前不拆成 `ansi-c-quoting`、`histfile`、`dev-tcp` 等小页，因为 raw 仍以短技巧集合为主。

## 常见陷阱

- 没确认 eval 层数就直接套 payload，结果引号、反斜杠和换行语义全错。
- 只追求弹 shell，忽略题目实际只需要读一个文件或内部服务。
- stdout 关闭时继续尝试普通输出，没检查 fd 0、stderr 和网络外带。
- rbash 场景只试 `cd`/`/bin/sh`，没检查 `BASH_ENV`、`PATH`、`LD_PRELOAD` 或可写目录。
- escape 后没有进行 post-shell 枚举。

## 关联技巧

- [pyjails.md](pyjails.md)
- [interactive-containers-jails-and-solvers.md](interactive-containers-jails-and-solvers.md)
- [source-backdoors-and-restricted-shell-tricks.md](source-backdoors-and-restricted-shell-tricks.md)
- [linux-privesc.md](linux-privesc.md)
- [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md)
- [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md)
- [misc-tooling.md](misc-tooling.md)

## 原始资料

- [bashjails.md](../raw/misc/bashjails.md)
