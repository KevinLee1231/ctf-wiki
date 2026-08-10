# Apache!

## 题目简述

目标使用 Apache HTTP Server，根路径提供静态资源，`/proxy` 路径启用了 `mod_proxy`。后端 flag 服务只在 `http://internal.host/flag` 上可达，需要利用 CVE-2021-40438 造成的 SSRF，让 Apache 代为访问内网地址。

## 解题过程

题目给出的虚拟主机配置表明，代理能力并不挂在站点根路径，而是配置在 `/proxy`：

```apache
<VirtualHost *:80>
    DocumentRoot "/usr/local/apache2/htdocs"

    <Location /proxy>
        ProxyPass https://www.google.com
    </Location>
</VirtualHost>
```

因此，网上针对“整个站点都是反向代理”场景的 payload 不能原样使用，入口必须保留 `/proxy` 前缀。将伪造的 Unix Domain Socket 描述与目标 URL 用 `|` 分隔，构造请求：

```text
/proxy?unix:a{5000}|http://internal.host/flag
```

Apache 在处理这段畸形代理 URI 时，会把分隔符后的 `http://internal.host/flag` 当作实际代理目标，由服务器从内网取回响应，flag 随响应正文返回。

CVE-2021-40438 的核心影响是：当攻击者能够影响转交给 `mod_proxy` 的请求 URI 时，受影响版本可能向攻击者选择的源站发起代理请求。题目给出的配置和 `/proxy` 路由决定了 payload 的落点；漏洞说明可参见 [Apache HTTP Server 2.4 漏洞公告](https://httpd.apache.org/security/vulnerabilities_24.html#CVE-2021-40438)。

## 方法总结

SSRF 利用不能只照搬公开 payload，还要结合真实代理配置判断哪个 URL 前缀会进入 `mod_proxy`。修复时应升级 Apache，并避免把未经严格解析和白名单校验的 URI 交给代理模块；内网敏感服务也不应仅依赖“外部无法直接访问”作为安全边界。
