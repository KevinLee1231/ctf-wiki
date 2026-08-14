# bi0sCTF 2024 - badNotes

## 题目简述

题目是一个 Flask 笔记站。用户可以注册、登录，并把 Base64 解码后的任意字节写入自己的上传目录；`/dashboard` 则由 Flask-Caching 的 `FileSystemCache` 缓存。缓存文件不是普通文本，而是“4 字节过期时间 + pickle 序列化对象”。

应用试图通过文件名黑名单阻止写入缓存目录，但只检查最终绝对路径中是否出现字符串 `caches`。利用 Linux 的 `/proc/self/fd/<n>` 可以在路径文本不含 `caches` 的情况下指向进程已经打开的缓存文件，再通过竞态覆盖它。下一次缓存读取会反序列化恶意 pickle，从而执行命令。

## 解题过程

### 确认两个可组合的原语

`/makenote` 的核心逻辑近似为：

```python
content = base64.b64decode(request.form.get("content"))
file = os.path.join(user_dir, title)
if "caches" in os.path.abspath(file):
    abort(400)
with open(file, "wb") as f:
    f.write(content)
```

`os.path.join` 不会消除 `../`，所以文件名可穿越用户目录。应用只禁止 `py`、`sh` 扩展名和 `*?[]`，并没有禁止斜杠或 `..`。另一方面，`/dashboard` 带有：

```python
@app.route("/dashboard", methods=["GET"])
@cache.cached(timeout=1, query_string=True)
def home():
    ...
```

不同查询参数会产生不同缓存键。连续访问 `/dashboard?a=1`、`/dashboard?a=2` 等 URL，可以让服务频繁创建、打开和替换缓存文件。

### 构造合法的文件系统缓存内容

Werkzeug 的文件系统缓存会先读取一个本机字节序的 32 位过期时间，再对剩余内容执行 `pickle.load`。因此不能只写一个 pickle；官方 exploit 按缓存格式拼接载荷：

```python
import os
import pickle
import struct
import time

class PickleRce:
    def __reduce__(self):
        return os.system, (COMMAND,)

expiry = struct.pack("I", int(time.time()) + 3600)
payload = expiry + pickle.dumps(PickleRce())
```

过期时间必须位于未来，否则缓存层会把文件当作失效项，不进入正常反序列化路径。

### 通过 `/proc/self/fd` 竞态覆盖缓存文件

缓存文件名不可预测，而且直接写 `../../../caches/...` 会触发字符串检查。不过 Linux 会把 `/proc/self/fd/n` 解析为进程文件描述符 `n` 当前指向的对象。只要写入发生时某个候选描述符恰好对应一个以写模式打开的缓存文件，就能绕开路径检查完成覆盖。

使用同一登录会话并行运行两条请求流：

1. 第一条不断以新查询参数请求 `/dashboard`，扩大缓存创建窗口；
2. 第二条循环提交标题 `../../../proc/self/fd/6` 到 `../../../proc/self/fd/14`，内容为上述缓存载荷的 Base64；
3. 命中正确描述符后，缓存文件被恶意 pickle 替换；
4. 再次访问相应缓存键，Flask-Caching 读取并反序列化该文件，触发 `os.system(COMMAND)`。

竞态是否命中可先用无害命令验证，例如在临时目录创建标记文件。确认执行后，再把命令替换为读取 flag 并回传到自己控制的接收端；具体回传地址不应写死在通用 exploit 中。

## 方法总结

本题不是单独的目录穿越或 pickle 反序列化，而是四个条件的组合：可控二进制文件写入、文件系统缓存使用 pickle、`/proc/self/fd` 提供路径别名、并发请求制造可写缓存描述符。路径黑名单检查的是字符串，而内核最终解析的是文件描述符所指向的对象，两者语义不一致，最终把普通笔记写入升级成了缓存反序列化 RCE。
