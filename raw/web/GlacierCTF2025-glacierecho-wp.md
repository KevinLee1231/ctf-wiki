# GlacierCTF 2025 GlacierECHO

## 题目简述

题目是 Django Echo 平台和一个登录管理员账户的 Chromium bot。`/echo` 只允许 `text/plain`，但校验使用 Werkzeug `parse_options_header()`，响应却原样写入用户给出的 Content-Type。构造 `text/plain;x=x,text/html` 时，Werkzeug 认为主类型仍是 `text/plain`，浏览器处理组合 MIME 值时却将响应当作 HTML，从而触发 XSS。

由于管理员 session cookie 是 HttpOnly，XSS 不能直接读取。预期链进一步利用 Unicode 空白 cookie 名与 Django `strip()` 的解析差异，让 bot 后续携带攻击者的 session 创建含 flag 的 Echo，最终在攻击者自己的历史记录中读取。

## 解题过程

### 1. 利用 Content-Type 解析差异获得脚本执行

服务端逻辑为：

```python
content_type = request.GET.get("type", "text/plain")
parsed_type = parse_options_header(content_type)[0]
if parsed_type != "text/plain":
    return HttpResponse(..., status=403)

response = HttpResponse(message + "\n" + message + "\n" + message)
response["Content-Type"] = content_type
```

本地对固定版本 Werkzeug 验证：

```python
parse_options_header("text/plain;x=x,text/html")
# ('text/plain', {'x': 'x'})
```

因此该值通过后端白名单，却被完整放入响应头。message 使用能在 HTML 文档中自动触发的 payload：

```html
<img/src/onerror=eval(atob('BASE64_JS'))>
```

先以普通 base station 身份调用 `/echo` 保存这条消息，刷新首页取得 Echo ID，再 POST `/report`。report 会按 ID 取出 Echo，并让管理员 bot 访问保存的 type 和 message。

### 2. 构造 Unicode cookie shadow

注册普通账户后，客户端已知自己的 `__Host-session` 值。嵌入 XSS 的 JavaScript 为：

```javascript
document.cookie = String.fromCodePoint(0x2000)
  + '__Host-session=<attacker-session>; Path=/;';
```

U+2000 EN QUAD 使浏览器把它视为另一个 cookie 名，因此不会覆盖真正的 HttpOnly 管理员 cookie，也不会对它应用精确名称 `__Host-` 的前缀约束。请求头随后同时含有管理员 cookie 和这个“前面多一个 Unicode 空白”的攻击者 cookie。

Django 5.0.1 的 `parse_cookie()` 对每个分号分段执行：

```python
key, val = key.strip(), val.strip()
cookiedict[key] = val
```

Python `str.strip()` 会删除 U+2000，于是两个名字在 Django 中都规范化为 `__Host-session`。在题目使用的 Chromium 流程中，新写入的同路径 cookie 后置于原 cookie；Django 逐项写字典时，攻击者值覆盖管理员值，SessionMiddleware 最终把该请求识别为攻击者账户。

### 3. 让 bot 主动把 flag 存入攻击者历史

bot 的固定流程为：

1. 登录 admin 并访问 `/control-center`；
2. 访问被举报的恶意 Echo，触发 XSS 并设置 shadow cookie；
3. 请求 `/echo?type=text/plain&message=Admin test: <flag>`。

第三步开始时 cookie 解析已被切换到攻击者 session。`echo()` 因而把包含 flag 的新记录写到攻击者 BaseStation ID，而不是管理员。exploit 只需用原 requests session 每隔数秒刷新首页，从自己的 `past_echoes` 中匹配：

```text
gctf{183A876Z_3verY0nE_L0v3333s_C00k!3s_847AHZ01}
```

## 方法总结

完整利用由两个 parser differential 串联：Werkzeug 与浏览器对畸形 Content-Type 的理解不同，浏览器与 Django 对 Unicode cookie 名的规范化又不同。XSS 本身读不到 HttpOnly cookie；它通过 cookie shadow 改变后续请求的服务端身份，再借 bot 的固定 flag 测试把秘密写入攻击者可见记录。修复应只使用解析、重建后的固定 `text/plain; charset=utf-8` 响应头，拒绝原始 type 中的逗号和异常参数，并在进入 session 框架前拒绝非 ASCII cookie 名及重复敏感 cookie。
