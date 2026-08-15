# rrrrrrrrrental

## 题目简述

租车页面通过无限滚动逐页加载车辆，浏览完全部页面后才允许访问隐藏功能。前端还注册了一个混淆 Service Worker：当浏览器请求真正的 `/features/feature.json` 时，它会悄悄改写到拼错的 `/features/faeture.json`，返回假 flag。解法需要同时满足服务端 Session 中的浏览页数条件，并绕开只存在于浏览器客户端的请求改写。

## 解题过程

### 分离服务端门槛与客户端欺骗

车辆接口每页返回 50 条数据，并把页号记入 Session：

```javascript
const LIMIT = 50;

if (!req.session.pagesViewed) req.session.pagesViewed = [];
if (!req.session.pagesViewed.includes(page)) {
  req.session.pagesViewed.push(page);
}
```

真实功能接口根据车辆总数计算页数，只检查当前 Session 已访问的不同页号数量是否足够：

```javascript
const totalPages = Math.ceil(cars.length / LIMIT);
if (!req.session.pagesViewed ||
    req.session.pagesViewed.length < totalPages) {
  return res.json({status: "error", message: "You must view all cars first."});
}
```

因此必须在同一个 `connect.sid` 下连续请求车辆页。另一方面，`main.js` 注册 `/sw.js`；将混淆代码还原后，核心逻辑等价于：

```javascript
self.addEventListener("fetch", event => {
  let request = event.request;
  if (request.method === "GET" &&
      request.url.toString().includes("feature.json")) {
    request = "/features/faeture.json";
  }
  event.respondWith(fetch(request));
});
```

注意两个文件名：

```text
/features/feature.json   -> 真 flag，受 Session 检查
/features/faeture.json  -> 假 flag，静态文件
```

Service Worker 只能拦截其作用域内的浏览器请求，并不能改变服务端路由。卸载或绕过它即可请求真路径。

### 保持 Session 直接请求后端

使用 `requests.Session` 比在浏览器中修改 Service Worker 更稳定：

```python
import requests

base = "http://TARGET:3000"
session = requests.Session()

page = 0
while True:
    response = session.get(
        f"{base}/static/cars.json",
        params={"page": page},
    )
    data = response.json()
    if data == ["done"]:
        break
    page += 1

result = session.get(f"{base}/features/feature.json")
print(result.text)
```

循环在第一次得到 `['done']` 时停止；此前的真实页面都已由同一 Session 访问。Python HTTP 客户端不执行 Service Worker，所以最后一次请求会原样到达 `/features/feature.json`，得到：

```text
shellmates{sw_c4ch3_c0n7r0ll3d_7h3_7ru7h}
```

浏览器路线同样可行：开发者工具中注销 Service Worker 或禁用其 fetch 处理器，再保持当前 Session 刷新真接口。但直接 HTTP 请求更容易排除缓存与旧 Service Worker 状态干扰。

## 方法总结

这道题叠加了两个彼此独立的状态层：服务端 Session 决定是否授权，Service Worker 决定浏览器最终请求哪个 URL。只绕过前端会被浏览页数检查拒绝，只滚完页面又会被客户端改写到假文件。分析现代 Web 题时应区分浏览器侧拦截器与服务器实际路由；保持 Cookie 的非浏览器客户端常能直接验证后端真实行为。
