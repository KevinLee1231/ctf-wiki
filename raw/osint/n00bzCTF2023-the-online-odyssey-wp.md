# The online Odyssey

## 题目简述

题面只给出用户名 `blackhat_abhinav`。需要从公开账号一路关联到地图评论和 Twitter，并读取图片的无障碍替代文本，而不是只观察图片像素。

## 解题过程

先搜索 `blackhat_abhinav`，通过其公开 GitHub 资料定位同名 Instagram 身份。Instagram 的线索指向一条 Google Maps 评论；评论文本中留下了另一个账号标识。继续搜索该标识，可关联到 Twitter 账号 `Abhinav78082932`。

检查该账号在比赛时间附近发布的图片推文。flag 不在图片画面中，而在作者为图片填写的 ALT 无障碍文本里；展开图片说明后可完整读取：

```text
n00bz{gr0tt4_1sl4nd_1s_4_n1c3_pl4c3_t0_v1s1t}
```

## 方法总结

跨站 OSINT 要保存每次身份跳转的共同标识，避免同名误关联。社交平台图片还应检查 ALT 文本、标题和描述等可访问性元数据，它们不会通过普通图像识别直接显示。
