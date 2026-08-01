# Enabled

## 题目简述

题目提供一个 Bash jail。脚本清空 `PATH`，禁用 `exec`、`command`、`type`、`hash`、`cd` 和 `enable` 等内建命令，过滤重定向符、斜杠、分号、`&`、`$`、括号与反引号，最后用 `eval` 执行仍属于 Bash builtin 的输入。flag 位于当前目录上级的 `flag` 目录中。

过滤器限制了外部程序和常见展开语法，但没有禁用 Bash 的目录栈及脚本加载内建命令。

## 解题过程

`pushd` 可以在不使用被禁用的 `cd` 的情况下切换目录，而且目录名本身不需要斜杠。先进入父目录，再进入 `flag`：

```bash
pushd ..
pushd flag
```

此时不能用 `cat`，但 `source` 是仍可用的内建命令。让 Bash 把 flag 文件当脚本读取：

```bash
source flag.txt
```

文件内容不是合法命令。Bash 的解析/执行错误会把触发错误的原始行带回输出，于是无需外部读取工具就泄露：

```text
byuctf{enable_can_do_some_funky_stuff_huh?_488h33d}
```

## 方法总结

- 核心技巧：用 `pushd` 替代被禁用的 `cd`，再利用 `source` 的报错回显读取文件内容。
- 识别信号：Bash jail 只禁用部分 builtins、清空 `PATH` 并靠字符黑名单过滤时，应系统枚举仍启用的目录、读取和加载类 builtins。
- 复用要点：突破 shell jail 不一定需要命令执行；让解释器解析目标文件并在错误信息中回显内容，同样构成读文件原语。
