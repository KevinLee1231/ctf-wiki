# UIUCTF 2023 What's for Dinner Writeup

## 题目简述

题目要求寻找旅行者 Jonah Explorer 的社交媒体账号。已知他刚到芝加哥，去过 West Loop 一家“joyful”的意大利餐厅，并喜欢记录旅行和评价食物。提示询问 “joy” 翻译成什么。

## 解题过程

`gioia` 在意大利语中表示“喜悦、欢乐”。将这一翻译与地区和餐厅类型组合检索：

```text
Gioia Italian restaurant West Loop Chicago
```

可以锁定芝加哥 West Loop 的 Gioia Chicago。接下来不只查看餐厅官网，还要检查用户生成内容。搜索该餐厅的评论时可找到 Jonah Explorer 的 Yelp 用户资料；其评论提到：

- 他是在芝加哥旅行期间到访 Gioia；
- 喜欢这里的食物和巧克力蛋糕；
- 会把吃过的食物发到 Twitter；
- 明确留下账号名 `@jonahexplorer`。

这个账号名把餐厅评论与题目所说的“知名社交网络”连接起来。访问 Jonah Explorer 的 Twitter/X 资料并检查其比赛期间发布的内容，可以看到包含 flag 的帖子：

```text
uiuctf{i_like_spaghetti}
```

## 方法总结

本题是一条跨平台身份关联链：语言提示确定餐厅，餐厅评论暴露社交账号，账号动态给出最终信息。OSINT 中应记录每一步的唯一连接字段；这里分别是 `gioia`、餐厅名称、评论者身份和 `@jonahexplorer`。这样即使原始社交帖子后来删除或平台链接失效，正文仍保留足以理解结论的证据链。
