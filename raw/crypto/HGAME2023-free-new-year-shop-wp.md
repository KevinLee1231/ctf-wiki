# 零元购年货商店

## 题目简述

题目是一个年货商店服务。用户信息被序列化后使用 AES-CTR 加密，并作为 token 交给客户端；只有用户名为 `Vidar-Tu` 的用户才能购买 flag，但注册或登录接口不允许直接使用该用户名。

问题在于服务只做 CTR 加密，没有为 token 提供消息认证。攻击者虽然不知道密钥，却能定向翻转密文字节，使解密后的用户名从一个已知的等长字符串变成 `Vidar-Tu`。

## 解题过程

CTR 模式将计数器生成的密钥流 $S$ 与明文异或：

$$
C=P\oplus S.
$$

如果已知原明文 $P$，希望对应位置解密为 $P'$，只需构造：

$$
C'=C\oplus P\oplus P'.
$$

服务解密时会得到：

$$
C'\oplus S=P'.
$$

先使用与目标用户名等长的 `aaaaaaaa` 登录并取得 token。根据服务的序列化格式，用户名从解码后 token 的第 9 字节开始，共 8 字节；逐字节异或原用户名与目标用户名的差分即可。token 还经过 Base64 和 URL 编码，所以修改前后要按相反顺序解码、编码：

```python
from base64 import b64decode, b64encode
import sys
from urllib.parse import quote, unquote

if len(sys.argv) != 2:
    raise SystemExit(f"usage: {sys.argv[0]} TOKEN_FOR_aaaaaaaa")

encoded_token = sys.argv[1]
raw = bytearray(b64decode(unquote(encoded_token)))

offset = 9
old_name = b"aaaaaaaa"
new_name = b"Vidar-Tu"

for index, (old, new) in enumerate(zip(old_name, new_name)):
    raw[offset + index] ^= old ^ new

attack_token = quote(b64encode(raw).decode(), safe="")
print(attack_token)
```

把生成的值替换到 cookie 的 token 字段，再访问购买接口：

```http
GET /buy?prod=flag HTTP/1.1
Host: challenge.example
Cookie: token=PASTE_THE_SCRIPT_OUTPUT_HERE
```

后端解密后看到的用户名已变为 `Vidar-Tu`，权限检查通过，返回：

```text
hgame{5o_Eas9_6yte_fl1p_@t7ack_wi4h_4ES-CTR}
```

## 方法总结

AES-CTR 只保证机密性，不保证完整性。只要攻击者知道某段明文及其位置，就能通过 $C'=C\oplus P\oplus P'$ 精确篡改解密结果。实际系统应使用 AES-GCM、ChaCha20-Poly1305 等认证加密方案，或至少对 nonce 与密文整体计算并验证可靠的 MAC；不能把“密文不可读”等同于“密文不可改”。
