# Top of the Charts

## 题目简述

题目页面正文没有 flag。标题暗示关注 HTTP 报文的“顶部”，实际答案被放在响应头 `flag` 中，使用 HEAD 请求即可直接查看。

## 解题过程

发送 HEAD 请求：

```bash
curl -I http://target/
```

或保留完整响应头：

```bash
curl -sS -D - -o /dev/null http://target/
```

在响应头中可见：

```text
flag: UMDCTF-{h3@d1ng_t0w@rd5_th3_l1ght}
```

浏览器开发者工具的 Network 面板也能查看同一字段；页面 HTML 中不存在另一层隐藏内容。

## 方法总结

分析 Web 响应时不能只看 body。状态码、重定向位置、Cookie、自定义响应头和请求方法都可能承载线索。HEAD 请求只取元数据，适合快速检查响应头，但若服务器没有正确实现 HEAD，也可用 GET 加 `-D -` 达到相同目的。
