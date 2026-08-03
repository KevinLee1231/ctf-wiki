# An Unlikely Partnership

## 题目简述

题面要求识别 LISA 的意外商业合作对象，是三题联动 OSINT 的第二题。上一题得到的 Threads 账号提供了新的职业社交平台入口；即使没有完成上一题，也可以用机构全称和站点限定搜索独立定位。

## 解题过程

查看 `longislandsubwayauthority` 的 Threads 个人资料，可见一个指向 LinkedIn 的链接。目标页面显示为 `LISA Transit`，地点是 Bay Shore, New York，简介以 Long Island Subway Authority（LISA）全称说明其虚构交通业务，这些字段共同确认账号归属。

公开连接列表不可直接浏览，但技能页面暴露了一条背书记录：名为 `UIUC Chan` 的用户为 LISA Transit 的技能提供了 endorsement。这就是“合作伙伴”身份的可见关联。打开 `UIUC Chan` 的 LinkedIn 资料，其职位是 Marketing Trainee，About 区域第一行直接写有：

```text
uiuctf{0M160D_U1UCCH4N_15_MY_F4V0r173_129301}
```

不依赖 Threads 的备选路径是搜索：

```text
site:linkedin.com "Long Island Subway Authority"
```

搜索结果同时出现 LISA Transit 与 UIUC Chan，并在摘要中显示二者的组织关系，可从 LISA 页面继续检查背书。官方五张图片均是上述个人资料、搜索结果和 Flag 文本的 UI 截图，没有额外视觉线索，因此全部转写为正文而未归档。

## 方法总结

- LinkedIn 的技能背书、推荐、经历和联系人公开字段都可能泄漏关系图，即使连接列表本身不可见。
- 联动题应保留前一题的稳定 pivot，但也应给出站点限定搜索等独立路径，避免整篇 WP 只写“沿上一题链接进入”。
- 识别同名实体时要组合机构全称、地点、简介和跨平台账号，而不是仅凭头像或搜索排序作结论。
