# GreyCTF 2025 A4 Toilet Paper Writeup

## 题目简述

题目提供一个 Express 用户系统。普通用户注册后可以更新资料，但 `/admin/dashboard` 只接受 `userPermissions` 为“没有任何可枚举键的对象”的用户。更新接口会强制把请求体中的 `userPermissions` 改成 `{ access: "" }`，表面上无法直接提交空对象。

漏洞来自字段顺序校验与字段赋值使用了不同的索引来源：校验时会过滤未知键，真正赋值时却遍历完整请求体。通过插入额外键，可以让合法字段被写进错误的配置项。

## 解题过程

接口希望按以下顺序处理七个字段：

```javascript
const configFields = [
  'username', 'email', 'toiletExperience', 'dateUpdated',
  'smell', 'temperature', 'userPermissions'
];
```

顺序检查使用 `bodyKeys.filter(key => configFields.includes(key))`，所以不在列表中的键不会影响检查结果。然而后续循环却对 `Object.entries(req.body)` 的每一项递增 `inputIndex`，然后执行：

```javascript
userConfig[configFields[inputIndex]] = value;
```

注册并登录后，按如下顺序提交资料：

```json
{
  "username": "asd",
  "email": "asd@asd.asd",
  "toiletExperience": "",
  "a": [],
  "b": [],
  "c": [],
  "dateUpdated": "2025-06-27T17:29:16.032Z",
  "smell": "123",
  "temperature": "123"
}
```

服务端会在末尾追加 `userPermissions`。过滤后的合法键顺序仍完全等于 `configFields`，因此通过第一道检查；但完整循环中的 `a`、`b`、`c` 把索引向后推了三位。三者是数组，对 `.trim()` 的调用会抛错并被内层 `catch` 忽略。轮到原始 `dateUpdated` 时，当前目标键已是 `userPermissions`，于是写入一个 `Date` 对象：

```javascript
userConfig.userPermissions = new Date(value);
```

`JSON.stringify(Date)` 是日期字符串，不等于反篡改检查禁止的 `'{}'`；而 `Object.keys(Date).length` 又等于 0，恰好满足管理员中间件。更新成功后保持同一会话访问 `/admin/dashboard`，即可得到：

```text
grey{lol}
```

## 方法总结

本题不是常见的原型污染，而是“校验视图”和“写入视图”不一致造成的字段错位。只要攻击者能插入被校验过滤、却仍参与赋值计数的键，就可以控制后续合法值落入哪个目标字段。`Date` 又同时满足“序列化结果非空对象”和“无可枚举键”这两个相互矛盾的权限检查，最终完成提权。
