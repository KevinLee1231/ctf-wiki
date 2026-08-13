# GreyCTF 2023 Login Bot

## 题目简述

应用允许用户让机器人代为登录并发布文章。管理员密码本身就是 flag。应用会把文章中的外部 URL 记录为站内 `/url/<id>` 跳转，但机器人的登录循环错误地复用了旧页面标记和重定向后的当前 URL，导致第二次登录请求把管理员凭据 POST 到攻击者站点。

## 解题过程

先向 `/send_post` 提交一篇内容包含自有接收端 URL 的文章。管理员机器人正常登录后发布文章，`sanitize_content` 会创建一条 `Url` 记录并把外链替换为 `/url/<id>`；响应同时回显当前 URL 记录数，可据此得到 `id`。

第二次调用 `/send_post`，把 `url` 参数设为刚才的站内跳转：

```text
/url/<id>
```

机器人首先请求 `/login?next=/url/<id>`。登录页包含 `bot_login` 标记，于是循环执行：

```python
resp = session.post(resp.url, data={
    "username": "admin",
    "password": FLAG,
})
```

第一次 POST 登录成功后，`next` 被认定为安全站内路径，响应跟随 `/url/<id>` 重定向到攻击者站点。漏洞在于循环中的 `content` 仍是最初登录页的内容，所以第二轮依旧认为需要登录；此时 `resp.url` 已经是外站地址，机器人便把 `username=admin` 和 `password=<FLAG>` POST 到该地址。

从自有接收端记录的第二次请求体中读取：

```text
grey{r3d1recTs_r3Dir3cts_4nd_4ll_0f_th3_r3d1r3ct5}
```

## 方法总结

站内跳转本身通过了 `next` 校验，但后续自动跟随重定向改变了请求的信任域。登录机器人必须在每次发送凭据前重新验证最终 URL 的源，并在成功登录后立即退出循环；不能用旧响应内容决定对新地址执行敏感操作。
