# Jojo Chat 2

## 题目简述

新版聊天服务把身份信息放入 Base64 token：

```text
<username>;<role>|<32-byte SHA-256 digest>
```

源码中的签名不是 HMAC，而是直接计算：

```python
hashlib.sha256(SECRET + username.encode() + b";user").digest()
```

其中 `SECRET` 长度固定为 40 字节。验证通过后，程序取第一个分号前的字段判断用户是否存在，却取最后一个分号后的字段作为角色。这两个解析规则与前缀密钥哈希共同构成了可利用的 SHA-256 长度扩展漏洞。

## 解题过程

### 确认 token 的解析差异

普通用户 token 解码后形如：

```text
alice;user|<digest>
```

连接逻辑分别执行：

```python
name = b64decode(token).split(b";")[0].decode()
role = b64decode(token).split(b"|")[0].split(b";")[-1].decode()
```

因此只要构造出能够通过签名验证的：

```text
alice;user<sha256-padding>;admin|<new-digest>
```

`name` 仍是已注册的 `alice`，而 `role` 会变成 `admin`。

### 执行 SHA-256 长度扩展

设原消息为 $M=\texttt{alice;user}$，已知摘要为：

$$
H=\operatorname{SHA256}(K\mathbin\Vert M)
$$

SHA-256 属于 Merkle-Damgård 结构。已知 $H$ 和密钥长度后，可以把 $H$ 作为新的内部状态，在不知道 $K$ 的情况下继续计算：

$$
\operatorname{SHA256}(K\mathbin\Vert M\mathbin\Vert
\operatorname{pad}(K\mathbin\Vert M)\mathbin\Vert\texttt{;admin})
$$

先注册普通账号并 Base64 解码 token，分离原始数据和二进制摘要：

```python
from base64 import b64decode

token = b"<registration token>"
decoded = b64decode(token)
original, digest = decoded.rsplit(b"|", 1)

print(original.decode())
print(digest.hex())
```

将两项结果交给 `hash_extender`。`--secret 40` 是源码中固定的密钥长度：

```bash
hash_extender \
  --format sha256 \
  --data 'alice;user' \
  --signature '<digest-hex>' \
  --append ';admin' \
  --secret 40
```

工具输出的 `New string` 是十六进制编码的：

```text
alice;user<sha256-padding>;admin
```

`New signature` 是继续压缩后的摘要。按服务端格式将二者重新组合并编码：

```python
from base64 import b64encode

new_string_hex = "<New string>"
new_signature_hex = "<New signature>"

forged = (
    bytes.fromhex(new_string_hex)
    + b"|"
    + bytes.fromhex(new_signature_hex)
)
print(b64encode(forged).decode())
```

把伪造 token 交给服务，签名校验成功；账号存在性检查仍命中 `alice`，角色检查则读取末尾的 `admin`。进入管理员功能后得到：

```text
N0PS{b3w4R3_0F_l3NgTh_XT3nS1on_4Tt4cK5}
```

## 方法总结

- 核心技巧：对 `SHA256(secret || message)` 执行长度扩展，再利用前后不一致的字段解析把追加内容解释为管理员角色。
- 识别信号：签名使用普通哈希拼接秘密前缀，而不是 HMAC；密钥长度可知；业务逻辑允许消息尾部继续追加字段。
- 复用要点：长度扩展后的消息必须保留 SHA-256 padding 原始字节，不能按普通文本重新编码。即使摘要能够伪造，也仍需检查被追加字段在业务解析器中是否确实可达；本题“首字段作用户名、末字段作角色”正好满足这一条件。
