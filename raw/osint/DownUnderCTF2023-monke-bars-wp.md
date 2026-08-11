# DownUnderCTF 2023 monke bars Writeup

## 题目简述

题面称作者即将发布名为 `monke bars` 的歌曲，要求从公开音乐平台找到它并收听 flag。原始线索依赖 SoundCloud 搜索，但官方仓库已经保留 MP3 和完整歌词，因此题解可以脱离可能失效的曲目页面复现。

## 解题过程

在常见音乐分享平台中搜索精确标题 `monke bars`，赛事期间只有一首同名目标曲目。歌曲中段的说唱直接按字符读出 flag；官方归档歌词对应片段为：

```text
D-U-C-T-F left curly bracket
smackit-hack-it-drop-that-packet
crack-this-track
right curly bracket
```

把口述的花括号还原，并按题面要求去掉连字符、空格且使用小写，得到：

```text
DUCTF{smackithackitdropthatpacketcrackthistrack}
```

## 方法总结

本题的 OSINT 步骤是根据题名选择音乐平台并定位唯一曲目，最终证据则来自音频内容。长期归档不应只写“去 SoundCloud 听歌”；将决定性歌词逐字转写到正文后，即使外部曲目被删除，解题过程仍然完整。
