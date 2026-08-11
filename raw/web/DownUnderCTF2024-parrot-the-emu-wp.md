# parrot the emu

## 题目简述

这是一个 Flask 页面：服务器把用户提交的 `user_input` 直接交给 Jinja2 的 `render_template_string` 渲染，随后将渲染结果回显。题目的决定性问题是服务端模板注入（SSTI），因此归入 Web，而不是按原始 `beginner` 标签归档。

目标是让模板表达式访问 Python 运行时环境并读取容器内的 `flag` 文件。源码中的实际 flag 为 `DUCTF{PaRrOt_EmU_ReNdErS_AnYtHiNg}`。

## 解题过程

`render_template_string(user_input)` 会把 `{{...}}` 当作 Jinja2 表达式执行。Flask 的 `config` 对象可作为模板上下文的一部分访问；沿着类构造函数的全局变量表即可取得已经导入的 `os` 模块：

```jinja2
{{ config.__class__.__init__.__globals__['os'].popen('cat flag').read() }}
```

该表达式的含义依次为：

1. 取得 `config` 的类及其 `__init__` 函数；
2. 取得函数的 `__globals__` 字典，从中取出 `os`；
3. 调用 `os.popen('cat flag')` 并读取标准输出；
4. 把输出嵌入响应页面。

提交后回显 flag：`DUCTF{PaRrOt_EmU_ReNdErS_AnYtHiNg}`。

## 方法总结

服务端模板只能接收受信任的固定模板，用户数据应作为模板变量传入并保持自动转义，不能把用户文本本身作为模板源码编译。审计 Flask/Jinja2 时，只要看到 `render_template_string`、`Template(...)` 等 API 接收可控数据，就应先验证 `{{7*7}}` 一类无害探针是否被求值，并评估其能否触及应用全局对象。
