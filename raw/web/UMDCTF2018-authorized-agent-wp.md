# UMDCTF 2018 - Authorized Agent

## 题目简述

站点拒绝普通访问者读取 `/flag`，提示需要成为“authorized agent”。认证依据不是账号或会话，而是 HTTP `User-Agent` 请求头。

## 解题过程

先访问常见信息入口 `/robots.txt`。返回内容明确给出：

```text
User-agent: Blackberry Bold 9780
Disallow: flag
```

服务端中间件也印证了这一机制：只有请求头精确等于 `Blackberry Bold 9780` 才返回隐藏内容，否则响应 `Not authorized`。因此直接构造请求：

```bash
curl -A 'Blackberry Bold 9780' http://target/flag
```

响应中的混淆字符串解码后为：

```text
Deny Everything: UMDCTF-{s3cr3t_@g3nT_m@nG}
```

所以 flag 是：

```text
UMDCTF-{s3cr3t_@g3nT_m@nG}
```

仓库 README 的 SHA-256 对应的是该 flag 加一个换行符后的字节序列：

```text
1df9f4c3458fd97282846456963423ceba83874cc2b86b3ec764cc88960a084e
```

## 方法总结

`robots.txt` 不是访问控制机制，却经常泄露路径、爬虫名称或题目所需请求头。本题的校验只是可伪造的客户端字符串比较；看到按浏览器或设备区分权限的逻辑时，应直接检查并重放相关 HTTP 头。
