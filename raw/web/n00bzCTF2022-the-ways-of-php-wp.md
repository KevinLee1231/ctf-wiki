# the ways of php

## 题目简述

题目先利用 PHP `DirectoryIterator` 对流包装器的支持枚举随机文件名，再利用“原始 HMAC-SHA256 后接 bcrypt”的错误组合绕过密码校验。两步分别解决入口定位和认证绕过。

## 解题过程

参数 `f` 被直接传给：

```php
foreach (new DirectoryIterator($c) as $f) {
    echo $f->getSize() . "\r";
}
```

令 `f=glob://*`，可用返回的文件条目数量作为前缀 oracle；逐字符尝试 `glob://<prefix>*`，恢复出登录页 `WzNsbej4VS.php` 和备份 `qHkldgoTQ4.tar`。

备份中的认证逻辑先计算二进制 HMAC，再交给 bcrypt：

```php
$hash = password_hash(hash_hmac('sha256', $real_password, 'GamingChair', true), PASSWORD_BCRYPT);
password_verify(hash_hmac('sha256', $input, 'GamingChair', true), $hash);
```

旧式 bcrypt 接口把输入视为以 NUL 结尾的 C 字符串。如果枚举一个候选密码，使其原始 HMAC 的首字节为 `\x00`，该输入和真实密码的预哈希都会在第一个字节处截断为空串，因而验证通过。用户名条件是 `md5(user)` 等于 `md5('admin')`，所以提交 `admin` 与任一满足首字节为 NUL 的候选密码即可得到：

```text
n00bz{D0nt_c0mb1n3_wh4t5_n0t_m34nt_t0_b3_c0mb1n3d}
```

## 方法总结

随机文件名并不能弥补目录枚举 oracle。密码层面也不应把含 NUL 的原始二进制摘要直接交给按 C 字符串处理的旧 bcrypt 实现；若确需预哈希，应使用安全的文本编码并遵循成熟的密码哈希方案。
