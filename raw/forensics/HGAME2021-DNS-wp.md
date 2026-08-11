# DNS

## 题目简述

附件是一份访问 `flag.hgame2021.cf` 时产生的网络流量。网页本身只会不断弹出“Flag is here but not here”，真正的 flag 不在 HTTP 响应正文，而藏在该域名的一条 DNS TXT 记录中。

## 解题过程

先在 Wireshark 中查看 DNS 查询，可以确认客户端依次请求了：

```text
A     flag.hgame2021.cf
AAAA  flag.hgame2021.cf
```

随后跟随 HTTP 流，响应正文的有效逻辑只有：

```html
<script>
while (true) {
    alert("Flag is here but not here")
}
</script>
<b>Do you know SPF?</b>
```

关键提示是 `SPF`。SPF 策略历史上通常通过 DNS 的 TXT 记录发布，因此不能只检查 A、AAAA 记录，而应主动查询该域名的 TXT 类型：

```powershell
nslookup
set type=TXT
flag.hgame2021.cf
```

返回的 TXT 内容为：

```text
hgame{D0main_N4me_5ystem}
```

域名如今是否仍可解析并不影响复现：题目流量已经给出了域名、网页提示与取证路径，最终 TXT 值也由官方 PDF 保留。

## 方法总结

网络取证不能只盯着应用层页面。网页中的短提示可能是在引导检查 DNS、证书或其他旁路元数据；本题的 `SPF` 明确指向 TXT 记录。处理类似题目时应同时梳理 DNS 查询类型、HTTP 内容和二者的时间关系，再决定 flag 实际位于哪一层。
