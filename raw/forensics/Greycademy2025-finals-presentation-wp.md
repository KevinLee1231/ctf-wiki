# Finals Presentation

## 题目简述

附件外层是 ZIP，内部是一份只有一张普通幻灯片的 PPTX。题目人为破坏了幻灯片母版布局的引用，使包含 flag 的布局不再出现在正常界面。需要把 PPTX 当作 OOXML 压缩包检查关系文件，恢复被孤立的布局并解码其中的 Base64 文本。

## 解题过程

先解开外层附件和 PPTX：

```bash
unzip finals_presentation.zip
mkdir unpacked
unzip "finals presentation.pptx" -d unpacked
```

`unpacked/ppt/slideLayouts/` 中同时存在 `slideLayout13.xml` 和 `slideLayout14.xml`。检查内容类型和母版关系可见，两处都引用 14，布局 13 因此成为未引用部件：

```text
[Content_Types].xml
  /ppt/slideLayouts/slideLayout14.xml

ppt/slideMasters/_rels/slideMaster1.xml.rels
  Target="../slideLayouts/slideLayout14.xml"
```

把这两处的 `slideLayout14.xml` 改回 `slideLayout13.xml`，删除多余的布局 14 及其关系文件，再重新打包即可在 Slide Master 中看到名为 `bottom text` 的布局。也可以直接搜索布局 13：

```bash
grep -oE '[A-Za-z0-9+/]{20,}={0,2}' \
  unpacked/ppt/slideLayouts/slideLayout13.xml
```

得到完整 Base64：

```text
Z3JleXtzaDB1bGRfaGF2M19zdGFuZGFyZDFzM2Rfc2wxZDNfdGgzbTNzX2F0X3RoM19zdGFydH0=
```

解码后为：

```text
grey{sh0uld_hav3_standard1s3d_sl1d3_th3m3s_at_th3_start}
```

内置缩略图的视觉核对显示，唯一普通幻灯片只是白底的 `Greycademy 2026!!!` 和 `bottom text`。解包布局的文字、坐标和媒体检查还确认一个写着 `grey{fake_flag}` 的诱饵布局；真正布局在页面下方放置上述完整 Base64 行。当前 PowerPoint/LibreOffice 渲染器无法稳定打开修复版，因而没有伪造 Slide Master 截图；普通封面和徽标也不承载解题信息，XML 文本已经完整转写。

## 方法总结

PPTX 本质上是由内容类型、关系图和 XML 部件组成的 ZIP。界面里“看不见”不代表数据被删除，应同时检查未被关系引用的 slide、layout、master、notes 和 media。恢复显示时必须同步修正内容类型与关系文件；若目标只是取证，直接读取孤立 XML 往往更稳妥。
