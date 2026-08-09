# Pastebin

## 题目简述

题目给出 Pastebin 用户名 `abhinav654321`，并强调内容创建于“很久以前”。当前页面只剩 `Nothing Here`，需要查询网页历史快照。

## 解题过程

访问该用户的公开 Pastebin 主页，找到唯一一篇 paste。当前版本没有 flag，因此把该 paste 的具体 URL 提交到 Wayback Machine 查询历史。

2024 年 6 月 17 日的快照保存了被后续替换的内容，其中可读到：

```text
n00bz{l0ng_t1m3_ag0_m34ns_w4yb4ck}
```

## 方法总结

“当前为空”和“历史上为空”不是同一结论。题面中的时间措辞直接提示网页归档；查询时应归档具体 paste URL，而不仅是用户主页。
