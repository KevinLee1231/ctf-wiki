# LazyDogR4U

## 题目简述

站点泄露的 `www.zip` 显示，`flag.php` 只有在 `$_SESSION['username'] === 'admin'` 时输出 flag。`lazy.php` 为了省事，把 `$_GET` 和 `$_POST` 的所有键注册为普通变量；过滤器只是从键名中删除一次 `SESSION` 等字符串，因此可以通过双写构造 `_SESSION`，覆盖整个会话数组。

## 解题过程

关键授权逻辑为：

```php
if ($_SESSION['username'] === 'admin') {
    echo $flag;
}
```

变量注册代码等价于：

```php
$filter = ["SESSION", "SEVER", "COOKIE", "GLOBALS"];

foreach (['_GET', '_POST'] as $_request) {
    foreach ($$_request as $_k => $_v) {
        foreach ($filter as $word) {
            $_k = str_replace($word, '', $_k);
        }
        ${$_k} = $_v;
    }
}
```

输入键 `_SESSESSIONSION` 删除中间的 `SESSION` 后恰好变成 `_SESSION`。PHP 查询字符串还支持用方括号构造嵌套数组，因此直接请求：

```text
/flag.php?_SESSESSIONSION[username]=admin
```

解析后，普通变量 `${'_SESSION'}` 被赋值为数组 `['username' => 'admin']`，覆盖原有超全局变量并满足严格比较。数组键不要写成 `['username']`，否则引号会成为键名的一部分；应使用 `[username]`。

源码中的 `testuser` 登录还存在 PHP 弱比较问题：数据库哈希以 `0e` 开头且后面全是数字，可用同样被解释为科学计数法零的 MD5 值（例如 `QNKCDZO` 的 MD5）通过 `==`。但该路线只能登录普通用户，不能满足 `admin` 的严格比较，所以与取 flag 无关。

## 方法总结

把请求参数动态注册成变量会让攻击者控制程序命名空间，尤其可能覆盖 `_SESSION`、`GLOBALS` 等安全边界。删除式黑名单还会产生典型的双写绕过。本题应优先沿“flag 输出条件 → 可控变量赋值”建立最短数据流，避免被不影响权限提升的弱比较支线带偏。
