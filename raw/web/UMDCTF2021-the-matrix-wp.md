# The Matrix

## 题目简述

题目站点会根据 HTTP `User-Agent` 判断访问者是否为机器人。普通浏览器看到的页面没有 flag，只有包含特定关键词的客户端才会进入隐藏路径。

## 解题过程

检查响应和源码可发现判断逻辑寻找大小写形式的 `bot`。修改请求头：

```bash
curl -i -A 'friendly-bot' http://target/
```

服务器会重定向或直接返回 `/the-matrix` 页面。跟随跳转：

```bash
curl -i -L -A 'friendly-bot' http://target/
```

页面中出现：

```text
UMDCTF-{r0b0t_r3b3ll!0n}
```

这里不需要伪造搜索引擎 IP；应用只依据可控的 `User-Agent` 字符串判断。

## 方法总结

HTTP 头属于客户端输入，不能承担真实身份认证。分析条件响应时，可先比较浏览器、`curl` 和自定义头的结果。本题的决定性变量是 `User-Agent`，用最小修改复现后即可定位隐藏路由。
