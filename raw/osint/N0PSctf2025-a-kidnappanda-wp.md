# A Kidnappanda

## 题目简述

这是 Bibzy 绑架案四题链的第一题。题目要求从失踪角色 Bibzy Clickbear 入手，找出绑匪用于交流的网站，并以完整 URL 作为 flag 内容。

## 解题过程

先搜索 `Bibzy Clickbear` 及页面中偶尔出现的拼写变体 `Bibzy Clickear`。Bluesky 上与绑架事件有关的公开内容把调查引向 WebTopia 人物 **Mondey Websnare**，并暴露了其常用账号名。

将同一账号名交给跨站用户名枚举工具 WhatsMyName，可以关联到一个 GitHub 账号。仓库当前页面没有直接公开邮箱，但 Git 提交的 `.patch` 视图会包含提交者身份；从历史提交中可以取得：

```text
mondeywebsnare@gmail.com
```

使用邮箱关联查询可以确认该地址注册了 Gravatar。Gravatar 头像地址以规范化邮箱的 MD5 为索引：先把邮箱转为小写并去除首尾空白，再计算：

```python
from hashlib import md5

email = "mondeywebsnare@gmail.com"
digest = md5(email.strip().lower().encode()).hexdigest()
print(digest)
```

输出为：

```text
6d8b34ca7de38dbe90d9b6cbf2020c55
```

对应的 Gravatar 公开资料中填写了个人网站。该网站是 [TopiaNews](https://topianews.com/)，首页也确实围绕 Bibzy 失踪案、Mondey Websnare 及各 Topia 的政治宣传展开，符合“绑匪在哪里交流”的题意。

按要求把网站完整 URL 放入花括号：

```text
N0PS{https://topianews.com}
```

## 方法总结

这条链的关键 pivot 是“社交账号名 → GitHub 历史提交 → 邮箱 → Gravatar 公开资料 → 网站”。用户名枚举只负责发现候选账号，真正能够把多个平台身份连起来的是提交邮箱及其 MD5 索引。最终网站的关键内容已经在正文中概括，读者无需依赖外链才能理解它为何是答案。
