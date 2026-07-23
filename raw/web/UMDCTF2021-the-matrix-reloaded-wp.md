# The Matrix Reloaded

## 题目简述

第二关依据 `Accept-Language` 返回不同内容。服务端期待冰岛语语言代码 `is`，普通浏览器的默认语言无法触发 flag 分支。

## 解题过程

发送自定义语言偏好：

```bash
curl -i -H 'Accept-Language: is' http://target/
```

也可以使用带权重的标准格式：

```text
Accept-Language: is-IS,is;q=0.9,en;q=0.5
```

只要头部包含服务端检查的 `is`，响应就返回：

```text
UMDCTF-{1c3l@nd1c_d0m1n@t!0n}
```

浏览器中可通过开发者工具重放请求并修改该头，效果相同。

## 方法总结

语言协商头通常只用于内容本地化，不应作为访问控制。测试条件页面时要系统检查请求方法、路径、Cookie 与常见协商头。本题无需更改系统语言，直接构造一条 HTTP 请求就能验证判断条件。
