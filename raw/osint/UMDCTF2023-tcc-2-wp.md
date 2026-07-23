# TCC 2

## 题目简述

第二题要求找到 TCC 官网中的隐藏问答页，并根据成员的公开信息回答四个问题。服务端会逐项校验答案，全部正确后返回 flag。

## 解题过程

先枚举站点公开元数据。仓库中的 `sitemap_092791284912.xml` 和页面链接可引向 `secret.html`，该页面要求回答：

1. TCC 最近一次 CTF 的名次；
2. 成员 `p1ku` 的雇主；
3. 成员 `bree` 最喜欢的题目方向；
4. 成员 `blub` 准备购买的礼物品牌。

逐项搜索公开证据：

- 在 CTFtime 查找 The Charizard Collective 最近一次参赛记录。赛事为 DawgCTF 2023，名次是 `145`。
- 官网公开的 `p1ku_resume.docx` 中写有 `Security Analyst, Leidos (August 2020 - Present)`，因此雇主填 `leidos`。
- Discord 历史消息中，`bree` 明确表示最喜欢 `misc`。
- Discord 中关于婚礼礼物的讨论给出登记信息线索；结合姓名和公开的 Amazon Wedding Registry，可确认储物容器品牌为 `Shazo`，填写 `shazo`。

提交内容为：

```text
145
leidos
misc
shazo
```

仓库中的 `server.js` 对这四个值进行不区分大小写的精确比较；答案不全时会通过 `correct-answers` 响应头返回正确项数量，四项都正确才发送 flag 页面。因此该响应头也可以用来逐项排查错误答案。

最终得到：

```text
UMDCTF{y0u_sur3_kn0w_h0w_t0_d0_y0ur_r3s3@rch_289723}
```

公开赛后记录保存了 Discord、CTFtime、简历和婚礼登记信息之间的关联过程；上文已经转写全部关键证据，来源见：[UMD CTF 2023 赛后复盘](https://dree.blog/blog/umd-ctf-2023/)。

## 方法总结

- 核心技巧：把站点枚举、公开简历、赛事记录和社交平台消息组合成多源验证。
- 识别信号：站点地图、异常命名的 XML、公开文档和响应头都可能是题目刻意留下的导航或反馈机制。
- 复用要点：每个答案都应保存原始证据和时间点；“最近一次”会随时间变化，复现时应以比赛期间的快照为准。
