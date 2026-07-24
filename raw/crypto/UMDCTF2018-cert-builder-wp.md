# UMDCTF 2018 - Cert Builder

## 题目简述

附件是 `openssl x509 -text` 风格的证书转储，包含版本、序列号、签名算法、颁发者、有效期、RSA 公钥和签名值，但缺少原始 DER/PEM。需要按 X.509 的 ASN.1 结构重建证书，并从重建结果中继续提取 flag。

## 解题过程

转储中的关键字段为：

```text
Version: 1
Serial: 13062737156104707249
Issuer/Subject: C=US, CN=*.ctf.science
Validity: 2018-04-11 22:41:45Z .. 2021-12-08 22:41:45Z
Public key: RSA 2048, exponent 65537
Signature: sha256WithRSAEncryption
```

X.509 v1 证书的外层结构是：

```text
Certificate ::= SEQUENCE {
    tbsCertificate       TBSCertificate,
    signatureAlgorithm   AlgorithmIdentifier,
    signatureValue       BIT STRING
}
```

根据转储重新编码各个 DER TLV。需要特别注意整数最高位为 $1$ 时补 `00`、UTCTime 的格式、`SET` 的排序，以及 `AlgorithmIdentifier` 是否带 `NULL` 参数。转储没有展示字符串类型和 RDN 分组方式，因此可枚举少量合理组合。

无需猜测哪个组合正确：用证书中的 RSA 签名做校验即可。对候选 `TBSCertificate` 计算 SHA-256，并检查：

$$
s^e \bmod n
$$

解码出的 PKCS#1 v1.5 `DigestInfo` 是否包含该摘要。匹配组合得到：

```text
TBSCertificate SHA-256:
7ce69ebd5b92d849fc85720f2ce9d3d0c493aa35b52e681ae4bfce05c78cc5cc

Certificate DER SHA-256:
a32401def3e25507f7674896b222e681cc6139603c99dc8121bbf21132bcacfd
```

把 DER 做 Base64 编码并加上 PEM 头尾后，正文中出现了刻意构造的路径：

```text
/pastebin/com/raw/iYQuKtM3/
```

将它还原为历史地址 `pastebin.com/raw/iYQuKtM3` 后，页面内容是一段 Base64；解码即可得到：

```text
UMDCTF-{Yay_You_Can_read_C3rts_N0w!}
```

外链只承担最后一段数据的托管作用，关键内容和解码结果已经完整记录在本文中，不依赖该页面继续存活。

## 方法总结

证书文本转储并不保存所有 ASN.1 编码细节，但签名本身提供了强校验条件。重建时应把“字符串标签、RDN 分组、算法参数”等不确定项限制为小范围枚举，再用原签名验证精确的 `TBSCertificate`，不要仅以 OpenSSL 能否解析作为成功标准。
