---
type: technique
tags: [web, ssti, eval, command-injection, context]
skills: [ctf-web]
raw:
  - ../raw/web/ruby-php-upload-and-ssti-rce.md
  - ../raw/web/xml-command-and-graphql-injection.md
updated: 2026-07-27
---

# Server-Side Expression and Command-Context Escape

## 适用场景

不可信输入进入模板、表达式、语言 `eval`、shell 或命令包装器；需要先识别当前语法上下文，再闭合边界并到达文件读、对象访问或命令执行。

## 识别信号

- 算术表达式、模板标记或引号/分隔符会改变响应。
- 错误信息暴露模板引擎、shell、语言运行时或参数拼接方式。
- 输入被黑名单过滤，但仍存在属性访问、管道、重定向或替代语法。

## 最小证据

- 一个无副作用 payload 能证明输入确实被二次解释。
- 明确输入位于字符串、参数、模板节点还是 shell token 中。
- 保存执行身份、工作目录和输出/盲执行通道。

## 解法骨架

1. 用常量表达式和语法错误识别解释器。
2. 逐步恢复当前 quoting、escaping 和拼接模板。
3. 先读取固定文件或调用无副作用函数，再推进到对象链/命令。
4. 若无直接输出，切换 timing、DNS/HTTP OOB 或落盘验证。

## 关键变体

- SSTI：从模板对象或 helper 进入文件/进程能力。
- Language eval：利用语言表达式、反射或内建函数。
- Shell context：按引号、参数和重定向规则构造边界逃逸。

## 常见陷阱

- 未确认上下文就堆叠 payload。
- 将应用返回的字符串误认为真正执行结果。
- 只测试空格/分号，忽略参数注入和无空格替代。

## 关联技巧

- [ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md)
- [xml-command-and-graphql-injection.md](xml-command-and-graphql-injection.md)
- [upload-polyglot-and-content-type-confusion.md](upload-polyglot-and-content-type-confusion.md)

## 原始资料

- [ruby-php-upload-and-ssti-rce.md](../raw/web/ruby-php-upload-and-ssti-rce.md)
- [xml-command-and-graphql-injection.md](../raw/web/xml-command-and-graphql-injection.md)
