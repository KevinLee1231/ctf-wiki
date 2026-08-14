# lorem ipsum

## 题目简述

附件是一份 24 页的 PDF，页面内容全部是重复的 Lorem Ipsum 占位文字。真正线索不在正文，而在文档元数据中，目标是定位真实 flag 并排除诱饵。

## 解题过程

直接阅读 24 页正文不会获得新信息，应先检查 PDF 元数据：

```bash
exiftool loremipsum.pdf
```

关键字段为：

```text
Title    : Lorem Ipsum is Fun!
Author   : rat2llibz
Subject  : here's the real one :P grey{l0r3M_1pSUm_3XifT0oL}
Keywords : nothing to see here; grey{f4ke_flag}
```

`Keywords` 明确包含 `fake_flag`，而 `Subject` 直接标注 “here's the real one”，因此真实答案是：

```text
grey{l0r3M_1pSUm_3XifT0oL}
```

页面预览与跨页抽查可确认 PDF 正文只是重复占位文本，没有影响解法的图片、注释或公式；无需把这些无关正文转写进 WP。

## 方法总结

文档取证不能只看可见页面。标题、作者、主题、关键字、创建与修改时间等元数据都可能承载线索；同时要结合语义判断诱饵，而不是看到第一个形似 flag 的字符串就停止。
