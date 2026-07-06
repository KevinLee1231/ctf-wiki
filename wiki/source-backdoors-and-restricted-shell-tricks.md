---
type: family
tags: [misc, family, source-backdoor, restricted-shell, bash, rvim, histfile, command-exec]
skills: [ctf-misc]
raw:
  - ../raw/misc/source-backdoors-and-restricted-shell-tricks.md
updated: 2026-07-06
---

# Source Backdoors and Restricted Shell Tricks

## 作用边界

题目给源码、受限 shell、编辑器 jail 或“看起来正常”的服务代码，真正入口是隐藏后门、特殊命令前缀、历史文件变量、解释器插件或工具自身逃逸能力。它通常比漏洞利用更像行为差异和功能滥用。

## 识别信号

- 源码中存在奇怪前缀比较、hex 字符串、维护函数、admin/debug 分支或 `system()` 包装。
- shell 禁止 `cat/less/head`，但允许启动 bash、vim、rvim、python/lua/ruby 插件或环境变量。
- 命令过滤是黑名单，且错误信息暴露真实 shell/编辑器能力。
- 输入加上某个前缀后程序行为突然变化。

## 最小证据

- 能指出后门触发条件，例如 `strncmp(input,"exec:",5)` 或 hex 编码后的 `exec:`。
- 受限 shell 题至少确认允许的二进制、环境变量和 shell 选项。
- 编辑器逃逸题先确认 `:version` 中是否有 `+python3`、`+lua`、`+ruby` 等扩展。

## 分流流程

1. 源码题先搜 `system`、`popen`、`exec`、`eval`、hex literal、debug/admin/maintenance。
2. 把可疑字符串解码后尝试最小命令，例如 `exec:id`，确认输出通道。
3. 受限 shell 文件读取优先尝试 `HISTFILE=/flag /bin/bash && history`、`bash -v flag.txt`。
4. rvim/vim jail 检查插件；若有 Python/Lua/Ruby，用内置解释器调用系统命令。
5. 最后把绕过方式写成最小可重放命令，不依赖交互记忆。

## 隐藏入口分支

| 入口类型 | 最小验证 |
|---|---|
| 源码隐藏后门 | 前缀、hex 字符串、维护函数触发 `system()`。 |
| HISTFILE trick | 用 bash history 读取禁止直接 cat 的文件。 |
| `bash -v` | verbose 模式会打印读取的脚本/文件行。 |
| rvim plugin escape | `:python3 import os; os.system("sh")` 类插件执行。 |

## 常见陷阱

- 只看主业务逻辑，忽略 debug/maintenance 分支。
- 看到 rvim 就认定不能执行命令，没有检查编译特性。
- 在受限 shell 中硬拼黑名单绕过，而不是找允许工具的副作用。
- 不保存最小命令，导致 writeup 只剩“交互里试出来”。

## 合并与拆分结论

该页不作为单一 technique 使用。它保留为 misc family，因为源码后门、HISTFILE/`bash -v`、rvim 插件逃逸和受限 shell 工具副作用的第一步证据不同，但都属于“显式漏洞利用前先确认隐藏功能或允许工具副作用”的分流入口。若后续某一类 raw 增多，应拆出更具体 technique；现阶段由本页连接 [bashjails.md](bashjails.md)、[interactive-containers-jails-and-solvers.md](interactive-containers-jails-and-solvers.md)、[pyjails.md](pyjails.md) 和 [scripts-and-obfuscation.md](scripts-and-obfuscation.md) 更合适。

## 关联技巧

- [bashjails.md](bashjails.md)
- [interactive-containers-jails-and-solvers.md](interactive-containers-jails-and-solvers.md)
- [pyjails.md](pyjails.md)
- [scripts-and-obfuscation.md](scripts-and-obfuscation.md)
- [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)
- [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md)

## 原始资料

- [source-backdoors-and-restricted-shell-tricks.md](../raw/misc/source-backdoors-and-restricted-shell-tricks.md)
