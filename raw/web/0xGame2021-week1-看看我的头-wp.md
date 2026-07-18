# week1看看我的头

## 题目简述

题目按顺序校验 HTTP 请求方法、`User-Agent`、`X-Forwarded-For`、查询参数和 POST 表单。满足前三项后会返回一段 Base64 编码的 PHP 逻辑，按源码构造完整请求即可获得 flag。

## 解题过程

根据页面提示逐步调整请求：方法必须为 POST；`User-Agent` 中要包含 `N1k0la`；`X-Forwarded-For` 中要包含 `127.0.0.1`。仓库源码对后两项只使用正则搜索，因此并不验证请求头是否可信或是否精确等于目标值。

通过这些检查后，将响应中的 Base64 解码得到核心逻辑：

```php
$a = $_GET['0xGame2021'];
$b = $_POST['X1cT34m'];
$d = $_POST['Pupi1'];
$c = 'welcome to the 0xGame2021';
if (md5($b) == md5($d) && $a === $c) {
    echo $flag;
}
```

这里不需要寻找 MD5 碰撞，因为代码没有要求 `$b` 与 `$d` 不同；给两个字段传入相同的普通字符串即可稳定满足哈希相等。完整请求可以写成：

```http
POST /?0xGame2021=welcome%20to%20the%200xGame2021 HTTP/1.1
Host: challenge
User-Agent: N1k0la
X-Forwarded-For: 127.0.0.1
Content-Type: application/x-www-form-urlencoded

X1cT34m=a&Pupi1=a
```

响应中得到：

```text
0xGame{http_pr0t0c0l_1s_int3r3sting:-)}
```

## 方法总结

本题的关键是把 HTTP 的不同数据位置对应到服务端变量：请求方法、请求头、GET 查询和 POST 表单缺一不可。原 WP 将相同输入误写成“MD5 碰撞”；真正最简条件只是 `md5($b) == md5($d)`，直接令 `$b=$d` 即可。`X-Forwarded-For` 也只能由可信反向代理写入，不能直接作为本地来源的安全依据。
