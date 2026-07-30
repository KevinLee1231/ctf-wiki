# NepCTF2026 web？re？ Writeup

## 题目简述

题目是一条多阶段 Web 攻击链：用户推荐功能存在二次 SQL 注入，可取得管理员凭据；管理员的 SVG 预览存在 XXE/SSRF，可访问仅监听本机的 PHP 服务；PHP 反序列化对象链最终到达可控 `include` 并取得代码执行；进入容器后，还需利用 SUID `xxd` 读取 root 配置和日志，触发内部 flag 程序。

## 解题过程

### 二次 SQL 注入取得管理员账号

注册普通用户后，推荐功能会截取用户名中 `@` 后的部分并拼入 SQL。注册如下用户名：

```text
x@' UNION SELECT 1, username, password FROM users--
```

数据先被正常保存，后续推荐查询再次使用该值时才触发注入，因此属于二次注入。响应会带出 `users` 表中的管理员用户名和密码。

### SVG XXE 访问内部 PHP

管理员页面允许上传并预览 SVG。XML 解析器允许外部实体，容器中还有 Apache/PHP 监听 `127.0.0.1:80`，可用：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY internal SYSTEM "http://127.0.0.1:80/">
]>
<svg xmlns="http://www.w3.org/2000/svg" width="800" height="200">
  <text x="10" y="40">&internal;</text>
</svg>
```

预览结果回显内部首页和 PHP 源码信息。

### PHP 反序列化到文件包含

内部服务读取 `land` 参数并执行：

```php
$ser = $_GET['land'] ?? 'O:4:"test":N';
@unserialize($ser);
```

`test::__destruct()` 根据 `key` 把 `f` 当作类名实例化或当作函数调用；对象图还会设置全局 `filename`/`file`，最终可到达：

```php
include($GLOBALS['file']);
```

预期路径是生成 `php://filter` iconv chain，使 `include` 在读取普通资源时合成一段 PHP 代码并执行。归档时没有保留数千字符、与目标编码环境强绑定的链；复现时用 PHP filter-chain generator 针对命令重新生成即可。

题目还有更短的非预期：上传的 SVG 文件本身可控，而 PHP 未设置 `open_basedir`，因此可以让 `include` 直接包含上传文件中的 PHP 内容。

### 从 www-data 到 flag

取得 `www-data` shell 后找不到明文 flag，但 `xxd` 带 SUID。利用它读取：

```bash
xxd /root/.bashrc
xxd /root/.bash_history
```

可得环境变量 `SEND_KEY`，以及 root 曾执行 `/usr/local/bin/sendthef1ag`。逆向该程序可知：它读取 `SEND_KEY`，解密 flag，并把结果请求到本机 80 端口。无需重新实现解密，只需复现合法运行环境：

```bash
export SEND_KEY='<从 .bashrc 恢复的值>'
/usr/local/bin/sendthef1ag
xxd /var/log/apache2/access.log
```

新的 Apache access log 条目中包含 flag。也可以利用 SUID `xxd -r` 覆盖 root 可控文件直接提权，但读取配置、调用原程序再查看日志已经足以完成题目。容器还意外保留了 `/proc/1/environ`，可作为另一条获取环境变量的非预期路径。

## 方法总结

本题每一阶段都依赖上一阶段扩大可见范围：二次 SQLi 给管理员身份，SVG XXE 给内网访问，PHP 对象注入给命令执行，SUID `xxd` 再突破文件权限。长链题应记录每一步获得的新能力，而不是只收藏最终 payload；这样即使某条 PHP filter chain 因环境差异失效，也能用上传文件包含等同级原语替换。
