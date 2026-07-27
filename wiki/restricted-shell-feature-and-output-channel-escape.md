---
type: technique
tags: [pwn, pentest, bashjail, restricted-shell, output-channel]
skills: [ctf-pwn, ctf-pentest]
raw:
  - ../raw/pwn/bashjails.md
  - ../raw/pwn/source-backdoors-and-restricted-shell-tricks.md
updated: 2026-07-27
---

# Restricted-Shell Feature and Output-Channel Escape

## 适用场景

Bash/rbash 或命令代理限制命令名、字符、路径、重定向或输出，但 shell expansion、环境变量、内建命令、工具副作用和已有文件描述符仍可组合出执行或数据外带。

## 识别信号

- 过滤发生在命令字符串而非 shell 语义。
- `$IFS`、glob、parameter/command expansion、here-string 或内建命令仍可用。
- 命令执行成功但 stdout 被丢弃，stderr、文件、历史或网络仍可观察。

## 最小证据

- 列出允许字符、内建、环境变量、工作目录和 fd。
- 用无副作用命令确认展开/执行阶段顺序。
- 至少一个稳定输出通道能传回受控标记。

## 解法骨架

1. 将过滤器输入、shell expansion 和最终 argv 分开记录。
2. 用变量、glob、切片或现有文件生成禁用字符/命令。
3. 优先利用内建和已安装工具的读写/编辑/插件能力。
4. 无 stdout 时切到 stderr、文件、HISTFILE、DNS/HTTP 或继承 fd。

## 关键变体

- Bash 字符/命令黑名单。
- rbash PATH、斜杠和重定向限制。
- Post-shell 输出封锁与副作用通道。

## 常见陷阱

- 本地交互 shell 与远端非交互模式不同。
- 只找到命令执行，没有可验证输出。
- 使用命令替换时输出被外层过滤再次处理。

## 关联技巧

- [bashjails.md](bashjails.md)
- [source-backdoors-and-restricted-shell-tricks.md](source-backdoors-and-restricted-shell-tricks.md)
- [restricted-runtime-object-graph-and-capability-recovery.md](restricted-runtime-object-graph-and-capability-recovery.md)

## 原始资料

- [bashjails.md](../raw/pwn/bashjails.md)
- [source-backdoors-and-restricted-shell-tricks.md](../raw/pwn/source-backdoors-and-restricted-shell-tricks.md)
