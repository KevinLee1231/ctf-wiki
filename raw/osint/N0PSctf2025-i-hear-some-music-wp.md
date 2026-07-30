# I hear some music

## 题目简述

题目提示 N0PStopia 的幕后成员中有人创作音乐，要求找到相关作品并从音频中听出一串字符，再用 `N0PS{...}` 包裹。

## 解题过程

N0PSctf 官方网站的团队信息列出了核心成员，其中 `algorab` 对应 Thibault Huillet。用真实姓名和昵称组合搜索：

```text
"Thibault Huillet" algorab music
```

可以定位到 YouTube 作品 [Her Grave Remains in Calm and Serenity and the Woods Are Breathing](https://www.youtube.com/watch?v=em4Y5--PubM)。页面显示它属于专辑 `The Memories of Lady Sunflower`，发布账号的其他专辑封面还使用了 N0PSctf 标志，因此能够把音乐作者与比赛团队身份交叉确认。

完整播放作品后，可以听到一段逐字符拼读的 32 位十六进制字符串：

```text
21b2a4131fa98bb867f31e934bfe19f3
```

题目只要求原样包裹这串字符，不需要再把它当作 MD5 等摘要继续破解：

```text
N0PS{21b2a4131fa98bb867f31e934bfe19f3}
```

原题解中的两张图片只是 YouTube 搜索结果和频道界面截图，所有有效文字已经转写到正文，因此不再保留。

## 方法总结

人物型 OSINT 应先从比赛官方团队页建立“昵称—真实姓名”映射，再用两者组合降低同名误报。找到媒体后还要用账号内容和 N0PS 标志确认归属，最后完整收听而不是只看标题或描述。外部视频承载了原始音频证据，因此保留直接链接，同时把作品名、专辑名和拼读结果完整写入正文。
