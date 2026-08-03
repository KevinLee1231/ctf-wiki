# Mr. Blue Sky

## 题目简述

题目给出 Bluesky 账号 `@mrbluesky1989.bsky.social`，要求从这个看似普通的个人主页中找到隐藏信息。这是 UIUCTF 2025 OSINT 连题的入口，考点不是图像取证，而是 Bluesky 特有的 Starter Pack（入门包）功能及其暴露的社交关系。

## 解题过程

打开目标主页后，个人简介只有复古音乐、猫和鸟等普通兴趣。继续检查主页各标签，可以看到由该账号创建的 Starter Pack `My Favorite Places on BlueSky`。Starter Pack 是一组由用户整理的账号和订阅源，原本用于帮助新人快速关注一批内容，但对 OSINT 而言也等价于公开了一份“目标认为相关的账号”清单。

浏览该 Starter Pack 的成员，末尾出现账号 `Mark the Bird Watcher`，句柄为：

```text
@birdwatching1290.bsky.social
```

其个人简介直接写有：

```text
uiuctf{y0u_dr0pp3d_y0ur_cr0wn_k1ngf15h3r_132323098}
```

官方解答还记录了一条非预期路径：Bluesky 的全局搜索会索引账号简介，因此直接搜索 `uiuctf`，再切换到用户结果，也能命中 Mark 的账号和 flag。这条捷径可以得到答案，但没有覆盖题目原本想展示的 Starter Pack 信息泄漏。

## 方法总结

- 核心技巧：遍历社交平台的非默认标签，利用用户维护的 Starter Pack 恢复公开关系链。
- 识别信号：目标主页内容很少，但题面强调“profile 中还有别的东西”，通常意味着列表、合集、关注关系或自定义订阅源。
- 复用要点：平台内搜索可能直接索引简介，是有效的交叉验证手段；完整复盘仍应记录从目标账号到关联账号的实际 pivot，而不能只写“搜 flag”。
