---
type: technique
tags: [pwn, pentest, web, source-audit, backdoor, debug]
skills: [ctf-pwn, ctf-pentest, ctf-web]
raw:
  - ../raw/pwn/source-backdoors-and-restricted-shell-tricks.md
  - ../raw/pentest/linux-privesc.md
updated: 2026-07-27
---

# Source Audit, Hidden Backdoor and Debug-Mode Discovery

## 适用场景

源码、部署脚本、补丁、历史配置或包装器中存在未公开入口、调试条件、硬编码凭据、命令分支或环境开关，可绕过预期攻击面直接获得能力。

## 识别信号

- 题目提供源码/补丁，但表面功能与实际路由/命令不一致。
- 隐藏参数、magic value、环境变量或调试 header 控制敏感分支。
- Git 历史、备份、注释或未调用函数保留旧入口。

## 最小证据

- 从外部输入到敏感 branch/sink 给出完整数据流。
- 明确触发条件、权限和部署环境是否满足。
- 用低副作用请求/命令验证隐藏路径真实可达。

## 解法骨架

1. 列出入口、配置加载、环境变量、路由注册和命令包装器。
2. 搜索 debug/backdoor/test/admin、magic constant、旧 API 与异常分支。
3. 对照部署文件和版本历史，确认代码确实进入当前构建。
4. 构造最小触发并验证，再连接凭据、shell 或提权路线。

## 关键变体

- 隐藏 HTTP/CLI route。
- 编译/环境变量启用 debug 功能。
- Git/备份中的旧凭据或被移除后门。

## 常见陷阱

- 找到死代码就当作可利用入口。
- 源码版本与部署二进制不一致。
- 触发 debug 模式却未验证权限边界实际改变。

## 关联技巧

- [source-backdoors-and-restricted-shell-tricks.md](source-backdoors-and-restricted-shell-tricks.md)
- [linux-privesc.md](linux-privesc.md)
- [restricted-shell-feature-and-output-channel-escape.md](restricted-shell-feature-and-output-channel-escape.md)

## 原始资料

- [source-backdoors-and-restricted-shell-tricks.md](../raw/pwn/source-backdoors-and-restricted-shell-tricks.md)
- [linux-privesc.md](../raw/pentest/linux-privesc.md)
