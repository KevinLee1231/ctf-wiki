# GreyCTF2022 - t00 f4st

## 题目简述

首页源码注释提示存在管理员入口。`admin.php` 先发送重定向到 `index.php`，但随后仍在响应体中输出白色 flag 文本。这是 Execution After Redirect：HTTP 重定向不会自动终止服务端代码。

## 解题过程

枚举到 `admin.php` 后，不让客户端自动跟随 302，直接查看原始响应：

```bash
curl -i --max-redirs 0 http://target/admin.php
```

响应头包含 `Location: ./index.php`，但响应体仍有：

```html
<p><span style='color: #ffffff;'>grey{why_15_17_571LL_ruNn1n_4Pn39Mq3CQ7VyGrP}</span></p>
```

浏览器会很快跳转且文字为白色，所以肉眼看不到；代理历史、开发者工具或禁止跟随重定向都能保留原响应。源码在 `echo` 后调用 `die()` 已经太晚，敏感内容此前就发送了。

题目附带的猫照片只是商店模板装饰，与漏洞或 flag 无关，因此不归档。

## 方法总结

`header('Location: ...')` 只设置响应头，不会停止 PHP 执行。授权失败或重定向后必须立即 `exit`，并且敏感数据不应先写入响应。测试时应检查原始 3xx 响应，而不只看浏览器最终页面。
