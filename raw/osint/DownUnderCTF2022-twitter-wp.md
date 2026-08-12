# DownUnderCTF 2022 twitter Writeup

## 题目简述

题面只给出一句线索：“南半球最大的 CTF”的 Twitter。目标是识别比赛官方账号，并检查账号公开资料中的异常文本。

## 解题过程

“南半球最大的 CTF”直接指向 DownUnderCTF，官方账号用户名为 `@DownUnderCTF`。打开账号主页后，不需要翻查图片像素或外部短链；比赛期间 flag 直接放在个人简介中。

简介内容说明 DUCTF 连帽衫上的吉祥物名叫 Ducky，对应完整 flag：

```text
DUCTF{the-mascot-on-the-ductf-hoodie-is-named-ducky}
```

## 方法总结

这是一道最小化 OSINT 定位题。题面先给组织识别线索，再用平台名限定信息源；确认官方账号后，应依次检查用户名、显示名、简介、置顶内容和外链。由于社交媒体简介会随时间变化，归档必须把当时的关键事实与 flag 写入正文，不能只保留一个可能失效或内容已改变的主页链接。
