# DownUnderCTF 2023 discord Writeup

## 题目简述

题面让选手查看比赛 Discord 的 `#rules` 频道，并收听支持队伍制作的等待视频。真正的答案被读成了 NATO 音标字母，题目同时规定 flag 使用全大写且不含空格。

## 解题过程

视频末尾给出的完整口述内容如下；把外部素材中的关键信息保留在题解内，就不再依赖活动结束后可能失效的 Discord 消息或视频地址：

```text
DELTA UNIFORM CHARLIE TANGO FOXTROT LEFT-CURLY-BRACKET
ROMEO ECHO JULIETT ECHO CHARLIE TANGO HOTEL UNIFORM MIKE ALPHA NOVEMBER INDIA TANGO YANKEE
ROMEO ECHO TANGO UNIFORM ROMEO NOVEMBER
TANGO OSCAR
OSCAR UNIFORM ROMEO
SIERRA UNIFORM PAPA PAPA OSCAR ROMEO TANGO
QUEBEC UNIFORM ECHO UNIFORM ECHO
RIGHT-CURLY-BRACKET
```

每个单词取其所代表的字母，例如 `DELTA UNIFORM CHARLIE TANGO FOXTROT` 对应 `DUCTF`。继续转换并去掉分隔空格，得到：

```text
DUCTF{REJECTHUMANITYRETURNTOOURSUPPORTQUEUE}
```

## 方法总结

题目的难点是找到频道中的目标媒体，并识别 NATO 音标字母这一表示方式。外链只负责提供原始口述内容；完整转写和字符映射已经写入正文，因此复现不再依赖临时活动链接。
