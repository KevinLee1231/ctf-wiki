# drm-1

## 题目简述

DRM 客户端携带过期的元数据 `meta.dat` 及其摘要 `hash.dat`。服务端把 32 字节秘密直接放在消息前面，验证

$$
\operatorname{SHA256}(K\mathbin\Vert\text{meta}),
$$

却把它当作 HMAC 使用。元数据解析器还会遍历所有逗号分隔字段，并让后出现的 `made`、`user` 覆盖先前值。这正好允许 SHA-256 长度扩展攻击。

## 解题过程

已知原始摘要、原始元数据和秘密长度 32。利用 SHA-256 的 Merkle–Damgård 结构，在不知道 $K$ 的情况下计算：

$$
\operatorname{SHA256}(K\Vert\text{meta}\Vert\text{padding}\Vert\text{append}).
$$

追加字段使用当前时间，并再次写入合法用户名：

```python
import time
import requests
import hlextend

base = "https://TARGET"
meta = bytes.fromhex(open("meta.dat", encoding="utf-8").read().strip())
old_digest = open("hash.dat", encoding="utf-8").read().strip()

append = f",made:{time.time()},user:daniel-kpdfgo".encode()
sha = hlextend.new("sha256")
forged_meta = sha.extend(append, meta, 32, old_digest)

url = f"{base}/unlock/{forged_meta.hex()}/{sha.hexdigest()}"
print(requests.get(url, timeout=10).text)
```

SHA-256 填充字节位于旧字段和追加字段之间，不妨碍解析器识别最后两个逗号字段。新时间满足 1000 秒有效期，服务端返回 DRM key、nonce 以及 flag：

```text
tjctf{wh0_n33ds_sp0t1fy_a7dfd3e5}
```

## 方法总结

- `SHA256(secret || message)` 不是安全的消息认证码；应使用标准 HMAC，例如 `HMAC-SHA256`。
- 长度扩展不仅要求可伪造摘要，还要确认应用解析器能容忍哈希填充并接受追加字段；本题的“后字段覆盖前字段”完成了第二个条件。
- 构造攻击时必须准确猜中秘密长度；这里源码明确把密钥截为 32 字节，因此无需盲枚举。
