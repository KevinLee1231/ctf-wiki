# UMDCTF 2025 - A Minecraft Movie

## 题目简述

用户可以创建包含 HTML 的帖子，再提交 UUID 让管理员 bot 访问。bot 会以 `admin`
登录、打开该帖子并点击“Dislike”。后端只有在管理员提交正数 `likes` 时才会设置
`likedByAdmin = true`；自己的任意帖子一旦获得该标记，`/me` 就返回 flag。

帖子经过 DOMPurify，不能直接执行 XSS。真正的链条是 DOM clobbering 加
URL-encoded 参数污染：用帖子中的命名元素控制 `window.sessionNumber` 的字符串值，
向固定的 `likes=-1` 前面插入一个 `likes=1`。

## 解题过程

前端点击按钮时执行：

```typescript
await ensureSessionNumber();

fetch(`${API_URL}/legacy-social`, {
  method: "POST",
  headers: { "Content-Type": "application/x-www-form-urlencoded" },
  body: `sessionNumber=${window.sessionNumber}&postId=${postId}&likes=${likes}`,
  credentials: "include"
});
```

`ensureSessionNumber()` 只在 `window.sessionNumber === undefined` 时请求真实编号。
浏览器会把带特定 `id` 的元素暴露成 `window` 的命名属性；`HTMLAnchorElement`
转换成字符串时又会返回其绝对 `href`。因此创建如下帖子内容：

```html
<a
  id="sessionNumber"
  href="https://example.com/?x=1&amp;likes=1"
>Steve</a>
```

DOMPurify 会删除危险脚本，但默认允许普通的 `a`、`id` 和 HTTPS `href`，所以该元素
仍能把 `window.sessionNumber` clobber 成锚点对象。管理员打开帖子后，
`ensureSessionNumber()` 看到它已非 `undefined`，不会覆盖它。

bot 点击 Dislike 时，函数参数仍为 `-1`，但实际请求体变成：

```text
sessionNumber=https://example.com/?x=1&likes=1&postId=<UUID>&likes=-1
```

后端使用：

```javascript
let { postId, likes: likesValue } = req.body;
if (Array.isArray(likesValue)) {
  likesValue = likesValue[0];
}
```

重复参数被解析为 `likes = ["1", "-1"]`，代码明确选择第一个值，因此管理员提交的
有效增量为 `+1`。`sessionNumber` 本身从未在 `/legacy-social` 中验证。

完整流程如下：

1. 注册长度至少 8 的账号并保存 session cookie；
2. 创建带上述锚点的帖子，记录返回的 UUID；
3. 把 UUID 提交给 admin bot；
4. bot 登录后点击 `#dislike-button`，实际为帖子增加 1 个 like；
5. 用自己的 cookie 请求 `/me`，后端发现该帖子 `likedByAdmin`，返回：

```text
UMDCTF{I_y3@RNeD_f0R_7HE_Min3S}
```

## 方法总结

本题不是传统存储型 XSS，而是利用净化后仍保留的命名元素改变全局变量解析，再借未做
URL 编码的模板字符串注入重复参数。防御时应避免把会被 DOM named properties 影响的
值挂在 `window` 上，对每个表单值使用 `URLSearchParams` 正确编码，并让后端拒绝重复
关键参数。更根本的是，后端不应信任客户端传来的 session number，也不应通过“数组取
第一项”静默解决参数歧义。
