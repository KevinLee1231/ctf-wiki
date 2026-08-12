# DownUnderCTF 2021 - OTWhat 2

## 题目简述

更新服务改用 P-256 ECDSA 对 URL 的 SHA3-512 摘要签名，并在页面公开 16 条历史更新记录。签名生成脚本对其中两条记录复用了同一个随机 nonce $k$；因此这两条签名具有相同的 $r$，足以恢复 $k$ 和 OEM 私钥 $d$，再为恶意更新 URL 生成合法签名。

审计表中的签名是“Base64 包裹的 ASN.1 DER 序列 $(r,s)$”，并不是 PEM。ECDSA 关系为：

$$
r=x(kG)\bmod n,\qquad
s=k^{-1}(z+rd)\bmod n,
$$

其中 $n$ 是 P-256 基点的阶，$z$ 是 SHA3-512 摘要最左侧 256 位。

## 解题过程

先解析页面表格，将每条 Base64 签名解码为 DER，再寻找重复的 $r$：

```python
from base64 import b64decode
from Crypto.Util import asn1

records = []
for url, encoded in audit_rows:
    r, s = asn1.DerSequence().decode(b64decode(encoded))
    records.append((url, int(r), int(s)))

seen = {}
for url, r, s in records:
    if r in seen:
        url1, s1 = seen[r]
        url2, s2 = url, s
        break
    seen[r] = (url, s)
```

页面调试输出和服务源码表明哈希为 SHA3-512。FIPS ECDSA 在 P-256 上只使用摘要最左侧的 256 位：

```python
from Crypto.Hash import SHA3_512

def z_of(url):
    digest = SHA3_512.new(url.encode()).digest()
    return int.from_bytes(digest, "big") >> 256

z1, z2 = z_of(url1), z_of(url2)
```

对两条签名分别写出
$s_1k=z_1+rd$ 和 $s_2k=z_2+rd$，相减后得到：

$$
k=(z_1-z_2)(s_1-s_2)^{-1}\bmod n,
$$

再代回任意一条签名恢复私钥：

$$
d=(s_1k-z_1)r^{-1}\bmod n.
$$

```python
n = 115792089210356248762697446949407573529996955224135760342422259061068512044369
k = (z1 - z2) * pow(s1 - s2, -1, n) % n
d = (s1 * k - z1) * pow(r, -1, n) % n
```

用恢复的标量构造 P-256 私钥，并按服务期望的 DER 编码签署恶意 URL：

```python
from Crypto.PublicKey import ECC
from Crypto.Signature import DSS
from base64 import b64encode

private_key = ECC.construct(curve="P-256", d=d)
signer = DSS.new(private_key, "fips-186-3", "der")
url = b"https://EVILCODE/"
signature = signer.sign(SHA3_512.new(url))

payload = {
    "url": url.decode(),
    "signature": b64encode(signature).decode(),
}
```

提交后标准验签通过，服务执行恶意更新并返回：

```text
DUCTF{27C3 Console Hacking 2010 (PS3 3p1c F41l)}
```

## 方法总结

ECDSA 的 nonce 必须对每条消息唯一且不可预测。两条签名出现相同 $r$ 是 nonce 重用的直接信号，可用简单模运算恢复完整私钥；哈希长于曲线阶位数时还必须按标准截取 $z$，否则公式无法匹配。审计日志不应公开足以关联消息和签名的敏感历史数据，而实现端应使用经过验证的确定性 ECDSA 或安全随机源生成 nonce。
