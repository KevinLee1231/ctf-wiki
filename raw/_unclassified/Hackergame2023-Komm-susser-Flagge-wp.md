# Komm, süsser Flagge

## 题目简述

三小问都要求向 HTTP 服务发送合法的 `POST` 请求并在请求体中提交个人 token，但入口前的 iptables 规则会检查 TCP/IP 包中的字节。需要分别利用 TCP 分段、错误的 TCP 头偏移计算和 IPv4 Options 绕过过滤。

HTTP 只是最终承载协议，决定性障碍是逐包理解并操纵 IP/TCP 报头与 netfilter 匹配语义；它既不是 Web 应用漏洞，也不是多主机渗透链，故归入 `_unclassified`。

## 解题过程

### 第一问：拆开 `POST`

第一条规则对每个 TCP 包做字符串匹配，只要某一包中出现 `POST` 就拒绝。TCP 是字节流，HTTP 方法并不要求在同一个数据段内发送，因此建立连接后分两次写入：

```c
const char *token = getenv("TOKEN");
char request[4096];
int request_length = snprintf(
    request, sizeof(request),
    "OST / HTTP/1.1\r\n"
    "Host: example.com\r\n"
    "Content-Length: %zu\r\n\r\n%s",
    strlen(token), token
);

write(fd, "P", 1);

int yes = 1;
setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &yes, sizeof(yes));
write(fd, request, (size_t)request_length);
```

`TCP_NODELAY` 用来降低两次小写入被 Nagle 算法合并的可能。服务端重组后仍看到完整的 `POST` 请求，而防火墙逐包检查时只看到 `P` 和以 `OST` 开头的两个片段。完整程序还应检查 `getenv`、`snprintf`、`setsockopt` 与两次 `write` 的返回值，并在必要时循环写完短写数据。

### 第二问：让 `u32` 算错 TCP 载荷位置

规则为：

```text
0 >> 22 & 0x3C @ 12 >> 26 @ 0 >> 24 = 0x50
```

第一个表达式从 IPv4 的 IHL 得到 IP 头长度，跳到 TCP 头；第二个表达式本想从 TCP Data Offset 得到 TCP 头长度，再检查载荷首字节是否为 `0x50`，即字符 `P`。

标准写法在 `12 >> 26` 后还应执行 `& 0x3C`。题目漏掉了掩码，TCP 的 Reserved 位会混入偏移。可在出站 mangle 阶段把 TCP 头第 100 位（第一个 Reserved 位）置一：

```nft
table ip filter {
  chain OUTPUT {
    type filter hook output priority mangle; policy accept;
    ip daddr SERVER_ADDR tcp dport SERVER_PORT @th,100,1 set 1
  }
}
```

随后发送普通 POST。错误偏移不再指向真实载荷，规则无法匹配 `P`。另一个非预期办法是复用第一问：首段只有一个载荷字节，而 `u32` 需要一次读取四字节；越过 `skb` 尾部会使匹配直接失败。

### 第三问：把允许串放入 IPv4 Options

第三条链只接受包前 50 字节内含 `GET / HTTP` 的包，其余全部拒绝。这连 SYN 都会检查，而 SYN 通常没有 TCP 载荷，不能靠 HTTP 正文满足条件。

IPv4 Options 位于固定头之后且会出现在握手包中，可以通过 `IP_OPTIONS` 夹带允许串：

```c
char opt[12] = "\x44\x0cGET / HTTP";
if (setsockopt(fd, SOL_IP, IP_OPTIONS, opt, sizeof(opt)) < 0) {
    /* 不同内核接受的 option kind 不同，应换 kind 重试。 */
}
```

这里首字节是 option kind，第二字节是总长度，之后才是目标字符串。设置成功后，SYN、握手后的 ACK 以及数据包都会带该 IPv4 option，于是规则在前 50 字节内命中 `GET / HTTP`；TCP 载荷仍可发送真正的 POST 与 token。

部分内核会解释 kind 68 的 Timestamp 内容并改写其中一位，出现 `GUT / HTTP`。可改用内核允许的其他 kind，或为 Timestamp 布局预留两字节：

```c
char opt[14] = "\x44\x0e..GET / HTTP";
```

## 方法总结

三问分别利用了“应用字节流不等于单个数据包”“偏移表达式必须正确屏蔽位域”和“过滤范围覆盖整个 IP 包而不只覆盖 TCP 载荷”。分析此类包过滤题时，应逐项确认匹配单位、越界读取行为、报头位域与可变长区域，再构造仍能被远端 TCP/HTTP 栈正常重组的请求。
