# 时光穿梭机

## 题目简述

题目要求先用历史报刊中的旧式英文拼写定位一座帝王墓，再把墓址映射到现代公园，最后通过地图街景识别公园对面的中医馆。关键不是泛搜关键词，而是把报刊名、日期、人物异译和现实地理位置串成可验证证据链。

## 解题过程

题面给出《伦敦新闻画报》及 1946 年 4 月 20 日等线索。对应的 [British Newspaper Archive 检索页](https://www.britishnewspaperarchive.co.uk/search/results/1946-04-20/1946-04-20?FreeSearch=&PhraseSearch=&SomeSearch=&AnySearch=&NotSearch=&SortOrder=2&FrontPage=&Region=&County=london%2C+england&Place=&NewspaperTitle=illustrated%2Blondon%2Bnews&PublicTag=&IssueId=BL%2F0001578%2F19460420%2F&ContentType=) 显示：

```text
Published: Saturday 20 April 1946
Newspaper: Illustrated London News
County: London, England
Title: THOUSAND YEARS: THE TOMB OF WANG CHIEN, BANDIT AND EMPEROR OF CHINA
Page: 16
```

`Wang Chien` 是威妥玛式或旧式转写下的“王建”。王建墓即成都永陵，现址为成都永陵博物馆、永陵公园一带。把历史实体转换为现代地名后，再查看公园周边地图与街景，题目要求的“公园对面的中医馆”招牌为“汉方堂”。

最终提交：

```text
moectf{han_fang_tang}
```

## 方法总结

历史 OSINT 的常见难点是名称漂移：同一人名可能出现旧式罗马化、现代拼音和中文名。应先用日期、报刊名、版面标题等稳定字段锁定史料，再做人物名映射；得到墓址后才进入地图核验。正文保留了史料标题与日期，因此即使原检索页面以后失效，仍能理解该链接提供的关键证据。
