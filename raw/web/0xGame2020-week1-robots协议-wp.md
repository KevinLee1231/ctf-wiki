# week1robots协议

## 题目简述

题目利用 `robots.txt` 泄露未在页面中提供入口的敏感路径。`robots.txt` 本质上是给网络爬虫读取的访问建议，不是访问控制机制；其中的 `Disallow` 规则对普通用户没有强制作用，反而可能暴露目录或脚本名称。

官方附件中的规则为：

```text
User-agent: *
Allow: /
User-agent: CTFer
Disallow: /flaaaggg.php
```

## 解题过程

访问站点根目录下的 `/robots.txt`，可以看到对名为 `CTFer` 的爬虫声明了禁止抓取 `/flaaaggg.php`。浏览器并不会受该规则限制，直接请求此路径：

```text
GET /flaaaggg.php HTTP/1.1
Host: <题目主机>
```

官方源码中 `flaaaggg.php` 会直接输出 `$flag`，因此响应正文即为 flag。

## 方法总结

- 核心技巧：读取 `/robots.txt`，跟进其中暴露但未被页面链接的路径。
- 识别信号：题目提示“爬虫”“robots”或页面入口极少。
- 复用要点：`Disallow` 只是爬虫约定，不能代替鉴权；实战检查时还应关注其中泄露的后台、备份和调试路径。
