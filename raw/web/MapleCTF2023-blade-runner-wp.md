# blade-runner

## 题目简述

应用允许注册和登录，管理员凭据对应 Redis 中的 `admin` 键。注册接口会把请求体字段复制到普通 JavaScript 对象，拒绝显式用户名 `admin`，但后续又通过 `obj.username` 和 `obj.password` 读取值。目标是覆盖管理员密码后登录，并从 `/joi` 的随机响应中取得 flag。

## 解题过程

利用 `__proto__` 设置对象原型，把用户名放到继承属性中：

```json
{
  "__proto__": {
    "username": "admin"
  },
  "password": "honk"
}
```

属性复制执行到 `__proto__` 时会触发传统原型设置器，而不是创建普通自有属性。注册接口针对请求中显式用户名的检查没有看到顶层 `username`，但随后访问 `obj.username` 时会从原型链得到 `admin`；`password` 则是正常的自有属性。最终应用把 Redis 中 `admin` 对应密码改成 `honk`。

使用用户名 `admin`、密码 `honk` 登录后访问 `/joi`。该路由会在五种回复中随机选择，flag 只是其中一种，因此保持会话并重复请求，直到响应包含：

```text
maple{blade_runner_2049_jf834gnc_0YFR343V8}
```

## 方法总结

原型污染不一定要污染全局 `Object.prototype` 才有价值；只要同一临时对象在校验和落库之间以不同方式判断属性，改变它自己的原型就足够。防护时应使用明确 schema 提取字段、拒绝 `__proto__`/`constructor`/`prototype`，并用 `Object.hasOwn` 判断必需属性。取得授权后还要观察响应随机性，避免把一次未出现 flag 误判为利用失败。
