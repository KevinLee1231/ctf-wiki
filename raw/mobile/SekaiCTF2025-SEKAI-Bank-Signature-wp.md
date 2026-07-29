# SEKAI Bank - Signature

## 题目简述

APK 中的 `/api/flag` 接口要求请求头 `X-Signature`。签名算法不是服务器私钥方案，而是把 APK 签名证书的 SHA-256 摘要直接作为 HMAC 密钥：

$$
\operatorname{HMAC\_SHA256}
\left(
\text{method}\parallel\text{"/api"}\parallel\text{path}\parallel\text{body},
\operatorname{SHA256}(\text{app certificate})
\right).
$$

APK 签名证书是公开材料，任何人都能从安装包提取，因此这个“密钥”没有秘密性。决定性障碍是 Android APK 签名身份被误当作共享密钥，故归入 `mobile`，而不是按仓库原目录归入 Reverse。

## 解题过程

### 1. 定位请求签名逻辑

反编译 APK 后，在网络层可看到：

```java
@POST("flag")
Call<String> getFlag(@Body FlagRequest request);
```

拦截器计算待签名字符串：

```text
POST/api/flag{"unmask_flag":true}
```

随后以应用签名证书 SHA-256 的原始 32 字节作为 HMAC key。

需要严格区分两个哈希：

- 证书摘要以十六进制显示；
- HMAC key 是 `bytes.fromhex(digest)`，不是摘要字符串的 ASCII 字节。

### 2. 从 APK 提取证书摘要

仓库官方 WP 给出的唯一签名者证书 SHA-256 为：

```text
3f3cf8830acc96530d5564317fe480ab581dfc55ec8fe55e67dddbe1fdb605be
```

该值可以用任意支持 APK Signature Scheme v2 的本地签名检查工具独立复现。证书、公钥和证书指纹都随 APK 分发，本来就不能承担保密用途。

### 3. 复现 HMAC 请求

请求体必须与签名时使用的紧凑 JSON 完全一致：

```python
import hashlib
import hmac
import json
import requests

digest = (
    "3f3cf8830acc96530d5564317fe480ab"
    "581dfc55ec8fe55e67dddbe1fdb605be"
)
body = json.dumps(
    {"unmask_flag": True},
    separators=(",", ":"),
)
to_sign = "POST/api/flag" + body
signature = hmac.new(
    bytes.fromhex(digest),
    to_sign.encode(),
    hashlib.sha256,
).hexdigest()

response = requests.post(
    BASE_URL + "/api/flag",
    data=body,
    headers={
        "Content-Type": "application/json",
        "X-Signature": signature,
    },
)
print(response.text)
```

若使用 `requests.post(..., json=...)`，库可能加入与预期不同的空格；最稳妥的方式是先生成 `body`，同一份字节既参与签名又作为请求数据发送。

官方记录得到：

```text
SEKAI{are-you-ready-for-the-real-challenge?}
```

## 方法总结

代码签名证书证明“这个 APK 由谁发布”，并不向客户端隐藏任何秘密。把证书摘要当 HMAC key，只能阻挡没有查看 APK 的用户，无法建立服务器端鉴权。

真正的服务端授权应依赖用户会话、设备注册后的不可导出私钥、挑战响应或服务器持有的秘密。即使采用 Android Keystore，也不能仅验证客户端能生成某个固定请求签名，还要绑定 nonce、账户、时效和服务器挑战，防止重放。
