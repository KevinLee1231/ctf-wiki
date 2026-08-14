# CakeCTF2021 Yoshi-Shogi

## 题目简述

题目提供一个 Rust 编写的本地将棋客户端。客户端把当前棋盘编码为 SFEN，请求远端 `/ponder` 接口，再完全相信 JSON 中的 `bestmove`。当远端返回 `resign` 时，客户端直接进入玩家胜利状态并把解密密钥改成 `win`。

flag 只在右键开启的 flag mode 且状态为 `PlayerWin` 时显示，因此无需真的战胜将棋引擎，只需伪造一次未认证的 HTTP 响应。

## 解题过程

### 逆向客户端与协议

从二进制字符串和公开 Rust 源码都能定位请求：

```text
http://yoshi-shogi.cakectf.com:15061/ponder
    ?position=<SFEN>&hand=<持驹>&move=w
```

官方 API 把参数交给 `gpsusi`，再返回形如 `{"bestmove":"7g7f"}` 的 JSON。通信使用明文 HTTP，没有签名、证书固定或服务端挑战。客户端的关键分支为：

```rust
if resp["bestmove"] == "resign" {
    *state = State::PlayerWin;
    *key = vec![119, 105, 110]; // b"win"
}
```

### 让目标域名指向本地伪服务

在 hosts 或 DNS 层把 `yoshi-shogi.cakectf.com` 指到本机，并在 15061 端口提供任意 `/ponder` 路径，始终响应：

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"bestmove":"resign"}
```

等价的最小 Flask 逻辑是：

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.get("/ponder")
def ponder():
    return jsonify(bestmove="resign")

app.run(host="0.0.0.0", port=15061)
```

先在准备界面右键 Yoshiking 开启 flag mode。若随机轮到电脑，客户端会立即请求；若先轮到玩家，走一步合法棋触发电脑回合即可。

### 进入胜利状态并解密

客户端收到 `resign` 后把密钥设为循环字节串 `win`，再结合内置 `delta` 和前两项状态 $(7,23)$ 逐字节恢复 flag。正常界面直接显示：

```text
CakeCTF{https://www.nicovideo.jp/watch/sm19221643}
```

URL 是 flag 字符串本身，不需要访问。出题人的[官方复盘](https://ptr-yudai.hatenablog.com/entry/2021/08/30/000015)也把解法概括为篡改客户端与引擎服务之间的通信；公开源码进一步给出了 `resign -> PlayerWin -> key="win"` 的完整因果链。

界面人物、对话框和棋子字体都只是装饰资源，文字化协议和状态分支后不再提供独立解题信息，所以不保留图片。

## 方法总结

- 游戏客户端不能把未认证的远端响应直接当作胜负事实，尤其不能据此派生解密密钥。
- 逆向网络型客户端时，应同时检查请求目的地、传输层保护、响应格式和状态机分支。
- 本题的最短链是 `右键开启 flag mode -> 伪造 bestmove=resign -> PlayerWin -> 使用 win 解密`。
