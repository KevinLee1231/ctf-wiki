# GreyCTF 2023 Microservices

## 题目简述

网关使用 Flask 选择要代理的微服务，并把原始查询字符串原样转发；管理服务则由 FastAPI 解析同一份参数。两个框架面对重复参数时选取的值不同，使攻击者能让网关路由到管理服务，同时让管理服务认为请求的并不是 `admin_page`，进而触发携带管理员 Cookie 的 SSRF。

## 解题过程

网关通过 `request.args.get("service")` 取得第一个 `service`，但转发时使用未经归一化的 `request.query_string`。管理服务再调用 `request.query_params.get("service")`，取得最后一个同名值。因此请求可写为：

```text
/?service=admin_page&service=home_page&url=http://home_page
```

解析链如下：

```text
Flask 网关：service = admin_page  -> 路由到管理服务
FastAPI 管理服务：service = home_page -> 绕过 admin_page 拒绝条件
管理服务：GET url，并附带管理员 cookie
```

`home_page` 会在收到正确管理员 Cookie 时返回 flag，所以 SSRF 的响应经管理服务和网关原样带回：

```text
grey{d0ubl3_ch3ck_y0ur_3ndp0ints_in_m1cr0s3rv1c3s}
```

## 方法总结

这是 HTTP 参数污染导致的跨框架解析差异。微服务边界上不应一边用解析后的单值做授权，另一边又转发原始多值查询串。网关应拒绝安全敏感参数的重复项，按统一规则重新编码转发参数，并限制后端发起请求的目标范围。
