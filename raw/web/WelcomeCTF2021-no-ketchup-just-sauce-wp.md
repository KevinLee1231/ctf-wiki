# No Ketchup, Just Sauce

## 题目简述

WelcomeCTF2021 的 No Ketchup, Just Sauce 展示一个“建设中”网站。目标页面要求回答安全问题，但答案没有直接显示；站点遗留的爬虫规则、版本注释和备份文件泄露了完整答案。

## 解题过程

先访问常见的 `robots.txt`：

```text
User-agent: *
Disallow: /reborn.php
```

进入 `/reborn.php` 后，HTML 注释写着当前版本为 2.2.3，并提示备份包含 2.2.2。按常见备份命名尝试：

```text
/reborn.php.bak
```

备份文件可以直接下载，其中保留了服务器端比较的明文：

```php
strcmp(
    $ketchup,
    'no ketchup, raw sauce -- too many calories, not good'
) == 0
```

把该完整字符串作为 `ketchup` 字段 POST 到当前 `/reborn.php`，通过比较后返回：

```text
greyhats{n0_k3tchup_r4w_s4uc3_892e89h89e}
```

## 方法总结

解题路径是 `robots.txt` 暴露隐藏路由、页面注释暴露版本差异、备份文件暴露源代码与答案。`robots.txt` 不是访问控制，部署目录中也不应保留 `.bak` 等可下载备份；即使页面本身没有代码执行漏洞，信息泄露仍足以绕过安全问题。
