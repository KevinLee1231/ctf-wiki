# MiniLCTF2020 - MITM_1

## 题目简述

本题沿用同一份中间人流量，要求找出伪造证书签发者的 Common Name。核心是定位 TLS Certificate 握手，区分证书的 issuer 与 subject，并读取 issuer DN 中的 `CN`。

## 解题过程

在新版 Wireshark 中可用：

```text
tls.handshake.type == 11
```

旧版解析器可能显示为 `ssl.handshake.type == 11`。展开 `Handshake Protocol: Certificate -> signedCertificate -> issuer -> rdnSequence`，对比正常证书后可见异常签发者：

```text
OU = DKY CA
O  = DKY LYK.
CN = Liuyukun CA
```

题目问的是 fake issuer 的 Common Name，因此答案是完整字符串 `Liuyukun CA`，不是站点域名或组织名。做标准 Base64：

```python
import base64
print(base64.b64encode(b'Liuyukun CA').decode())
```

得到：

```text
TGl1eXVrdW4gQ0E=
```

提交格式为 `minil{TGl1eXVrdW4gQ0E=}`。

## 方法总结

X.509 的 DN 中，`CN` 是 `commonName`，但 issuer CN 与 subject CN 含义不同：前者标识签发者，后者通常标识证书持有者或域名。题面同时出现 “fake issuer” 与 “Common Name” 时，应在 issuer 分支读取 CN，不能只搜索任意一个证书 CN。
