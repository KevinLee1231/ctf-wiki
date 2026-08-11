# pkijs<

## 题目简述

服务要求上传 CMS `SignedData`：封装内容必须恰为 `I can forge a signed message!`，并且 `pkijs` 的证书链验证必须返回 `true`，才回显 flag。题目固定使用存在证书链校验问题的 `pkijs` 3.0.15；决定性障碍是 X.509/CMS 信任链绕过，归入 Crypto。

## 解题过程

服务端实际校验如下：只接受 `SIGNED_DATA`，取出封装内容比较固定字符串，然后以题目根证书为唯一 `trustedCerts`、开启 `checkChain`，调用 `signedData.verify`。所以只提交任意文本或伪造 flag 都无效，必须构造能被缺陷版本接受的 CMS。

官方求解器的构造方式是：

1. 新建攻击者 RSA 密钥和证书，`subject` 自定，但把证书的 `issuer` DN 设为题目根证书的 `issuer` DN；实际签名仍由攻击者私钥完成，因此它并非根证书签发。
2. 用攻击者私钥对固定内容做 RSA PKCS#1 v1.5 / SHA-256 签名，并以攻击者证书公钥的 SHA-1 `subject_key_identifier` 作为 CMS signer 标识。
3. 在 `certificates` 集合中依次放入攻击者证书和题目根证书，再提交生成的 DER CMS 到 `/upload`。

旧版本的链构建/证书识别缺陷使这份“issuer 名称伪装但并未由根私钥签发”的链被错误接受。求解器附带的 `win.bin` 就是上述结构的可复现制品。服务返回：

```
DUCTF{nice_splice_sice_a69bdb8eb2ca9e1}
```

## 方法总结

PKI 验证不能只依赖主体或 issuer 的名称相等；真正的授权关系必须落到签名验证、证书唯一性和链路径约束。审计 CMS 题时，要同时检查封装内容、signer 标识、证书集合和可信根配置。这里 HTTP 上传只是载体，能否通过取决于证书链语义，因此不应按 Web 题分类。
