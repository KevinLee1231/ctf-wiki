# Hip With the Youth

## 题目简述

题面称 Long Island Subway Authority（LISA）正在尝试社交媒体，并明确从 Instagram 开始。这是三题联动 OSINT 的第一题，目标是沿同一实体在不同平台之间的公开账号关联找到 Flag。

## 解题过程

先在 Instagram 搜索机构全称对应的紧凑用户名 `longislandsubwayauthority`，可定位显示名为 `LISA`、头像为蓝底黄色列车图标的账号。账号公开的两张列车图片本身没有隐藏信息，真正的 pivot 是个人资料中与 Instagram 同名绑定的 Threads 入口。

进入同名 Threads 账号后，查看其第一条帖子和回复链。主帖内容是“Trying out this threads thing, heard it's better than Twitter!”；账号随后自我回复称，如果帖子含 Flag 可能获得更多互动，下一条回复直接给出：

```text
uiuctf{7W1773r_K!113r_321879}
```

归档时没有保留官方的三张平台截图：它们只承载用户名、平台跳转和上述纯文本，相关证据已完整转写，截图中的社交 UI 不影响复现。需要注意，Threads 在部分地区可能不可访问；这属于平台可用性限制，并非解题还需要额外的图像分析。

## 方法总结

- 从题面给出的机构名建立稳定用户名，再核对显示名、头像和简介，避免把相似账号当作目标。
- 社交平台的账号绑定、个人资料链接和同名账户是常见 OSINT pivot；找到主页后不能只检查帖子图片，还要检查关联平台与回复链。
- 对易失的社交证据，应在 WP 中记录用户名、关键原文和最终值，不能只留下一个可能失效的外链或一张缺乏文字说明的截图。
