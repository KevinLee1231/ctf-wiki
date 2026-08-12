# DownUnderCTF 2020 - Web Badmin

## 题目简述

网页只显示游戏服务已经故障，页面源代码中却注释掉了一个指向 `epicgame.play.duc.tf` 的链接。该主机没有可用的 A/AAAA 记录并不代表名字不存在；题目把 flag 放在这个域名的 DNS TXT 记录中。

## 解题过程

查看 HTML 源码可以找到：

```html
<!-- <a href="https://epicgame.play.duc.tf">Play our new game here!</a> -->
```

浏览器访问失败通常只会尝试把主机解析为 IP 地址。应继续枚举其他 DNS 记录类型，尤其是常用于域名验证和文本发布的 TXT：

```bash
dig +short TXT epicgame.play.duc.tf
```

Windows PowerShell 也可使用：

```powershell
Resolve-DnsName -Name "epicgame.play.duc.tf" -Type TXT
```

比赛期间 TXT 响应直接包含：

```text
DUCTF{wait_im_confused_what_are_record_types_in_DNS???}
```

历史域名记录可能已经撤销，但仓库保留的官方 writeup 和 `flag.txt` 对该结果进行了交叉确认。

## 方法总结

DNS 名称可以同时拥有多种记录；A/AAAA 缺失或网页打不开，只能说明不能按常规方式连接 Web 服务，不能证明 TXT、MX、CNAME 等记录不存在。OSINT 式线索枚举在本题只是一个 Web 源码泄漏后的短步骤，起点仍是查看页面源代码中的隐藏主机名。
