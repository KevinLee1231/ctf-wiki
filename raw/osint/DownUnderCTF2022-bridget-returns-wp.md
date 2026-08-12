# DownUnderCTF 2022 Bridget Returns! Writeup

## 题目简述

题目给出地点字符串 `download.pausing.counterparts`，要求回答会面地点所在桥梁的名称。三个由句点分隔的普通单词符合 what3words 地址格式。

what3words 把地表划分为约 $3\text{ m}\times3\text{ m}$ 的方格，并在每种语言的地址体系中用三个单词标识方格。它只负责给出精确落点，桥名仍需通过地图标签和周边地理信息确认。

## 解题过程

在 [what3words 地址页](https://what3words.com/download.pausing.counterparts) 输入完整的三个单词，页面解析出的坐标约为：

```text
-27.267909, 153.076556
```

将坐标放入普通地图后，落点位于 Queensland 的 Clontarf 附近，并且明确落在跨海桥面上。检查桥梁标签和两端道路，名称为：

```text
Ted Smout Memorial Bridge
```

题目要求桥名且不区分大小写，可提交无空格形式：

```text
DUCTF{TedSmoutMemorialBridge}
```

## 方法总结

遇到“三个单词加句点”的地点线索，应优先识别 what3words，而不是把它当作域名或自然语言关键词。解析后还必须检查坐标实际落在哪个地物上；本题问桥名，所以城市 `Clontarf` 只是中间定位信息，不能作为最终答案。
