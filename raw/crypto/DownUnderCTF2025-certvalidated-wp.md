# certvalidated

## 题目简述

服务端固定要求对字符串 `just a random hex string: af17...74601` 签名，并接收一个 Base64 编码的 CMS `SignedData`。它调用 `endesive.plain.verify(content_info, message, [DUCTF_ROOT_CA])`，只有返回的 `hashok`、`signatureok` 与 `certok` 全部为真才输出 flag。题目镜像固定安装 `endesive==2.18.5`，并提供根证书。

关键点不在 RSA 私钥分解，而在该版本的 CMS 证书验证与签名者证书绑定不足：官方解题脚本可以让攻击者自己的公钥承担消息签名，同时把攻击者证书包装成看似由根 CA 签发的证书。

## 解题过程

### 关键观察

官方 `solve/solv.py` 生成一对攻击者 RSA 密钥和证书。证书的 `issuer` 被写成根证书的 `issuer`，但其公钥仍是攻击者公钥；脚本随后把攻击者证书 DER 中的 `signature` 字节串替换为根证书中的签名字节串。CMS 的 `SignerInfo.sid` 使用攻击者证书的 issuer 和 serial number，消息签名则由攻击者私钥生成。

因此三个校验所绑定的对象被拆开了：

- CMS 内容仍是服务端给出的原始字符串，所以摘要校验可以通过；
- `SignerInfo` 找到的证书包含攻击者公钥，故攻击者产生的 PKCS#1 v1.5/SHA-256 签名可以通过消息签名校验；
- 构造后的证书在题目所用验证路径中又被当作可由给定根证书信任。

这里不能把替换后的证书当成正常、可移植的 X.509 证书。它依赖的是题目锁定的 `endesive` 验证行为；换用严格的证书链验证器不应预期得到相同结果。

### 构造 CMS

下面是官方脚本的关键构造，省略了 `asn1crypto.cms.SignedData` 的固定字段填充。`my_crt.signature` 与 `root_crt.signature` 都是 RSA-2048 签名字节串，故脚本以字节替换的方式保留了 DER 长度。

```python
my_crt, my_key = create_cert(
    "my cert",
    pubkey=None,
    issuer=root_crt.issuer,
    issuer_privkey=None,
)

my_crt_der = my_crt.public_bytes(serialization.Encoding.DER)
root_crt_der = root_crt.public_bytes(serialization.Encoding.DER)
patched_crt = my_crt_der.replace(my_crt.signature, root_crt.signature)

sig = my_key.sign(to_sign, padding.PKCS1v15(), hashes.SHA256())

# SignedData.certificates 只放 patched_crt；
# SignerInfo.sid = (my_crt.issuer, my_crt.serial_number)，signature = sig。
cms_blob = build_signed_data(to_sign, patched_crt, my_crt, sig)
send_base64(cms_blob)
```

`create_cert` 的一个容易忽略的细节是：调用时虽然指定了根证书的 issuer 名称，却没有根私钥；函数会回退为用新生成的攻击者私钥对该证书签名。这正是后续替换证书签名字节所要掩盖的不一致。

### 验证

官方解题脚本读取服务端实际给出的待签名字符串，再发送上述 CMS blob；脚本的交互结果应使三个布尔值同时为真。题目配置给出的验证材料为 `DUCTF{d1d_y0u_just_f1nd_an_0day???}`。本归档只做了静态源码与官方解题脚本对照，未连接已关闭的比赛服务。

## 方法总结

- 核心技巧：利用 CMS 中“消息签名者证书”和“受信任 CA 关系”没有被正确绑定的实现缺陷，伪造一个由攻击者密钥签名的 `SignedData`。
- 识别信号：服务端直接把攻击者提供的 DER/CMS 交给某个库验证，并且只根据多个布尔返回值决定授权时，应审计证书链、`SignerInfo.sid` 与消息签名公钥是否指向同一可信对象。
- 复用要点：证书格式字段能伪造不代表任意实现都会接受；应固定库版本并用真实验证调用确认。构造中同时要保持内容、签名者标识与 ASN.1 长度一致，不能只替换可见的 PEM 文本。
