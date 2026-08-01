# GlacierCTF 2025 findme_v2

## 题目简述

附件是一个三页 PDF。正常阅读只能看到 Lorem ipsum 和 GlacierCTF 标识，页面资源里也没有可直接导出的图片。生成脚本却先把 flag 绘制到 PNG，再通过 PyMuPDF 新建 xref、写入空对象字典并调用 `update_stream` 保存原始 PNG 字节；这个对象从未被任何页面、XObject 或内容流引用。

所以 flag 不在可见页面中，而在 PDF 交叉引用表里的“孤儿对象”中，归类为隐写。

## 解题过程

### 1. 先确认可见层没有差异

对 `challenge/chall.pdf` 的 3 页逐页渲染，并与生成前的 `base.pdf` 对照。三页版式、文字和标识均一致，没有隐藏在裁切区、透明图层或页面外的可见 flag。文本提取也只返回页面上的占位文本。

这一步很重要：不能因为 PDF 里存在可疑流，就把它误说成“页面内隐藏图片”。实际证据是该对象根本没有进入页面对象树。

### 2. 遍历 xref 并识别文件签名

`chall.pdf` 的 xref 长度是 131，而基准文件为 130。新增的 xref 130 对象字典只有 `/Filter /FlateDecode` 和 `/Length`，没有 `/Subtype /Image`。PyMuPDF 会在 `xref_stream()` 中自动解开 Flate，返回值以 PNG 签名 `89 50 4e 47 0d 0a 1a 0a` 开头。

官方解法可以直接按已知编号提取：

```python
import fitz

doc = fitz.open("chall.pdf")
data = doc.xref_stream(130)
assert data.startswith(b"\x89PNG\r\n\x1a\n")
open("recovered-flag.png", "wb").write(data)
```

若不知道编号，更稳妥的做法是遍历 `range(1, doc.xref_length())`，对每个可读取 stream 检查文件 magic，并进一步检查该 xref 是否被任一页的对象引用。不要只依赖 `/Subtype /Image`，因为本题刻意没有设置这个类型。

### 3. 查看提取出的 PNG

提取出的 PNG 只有题名与可复制的 flag 字符串，不依赖颜色、布局或其它视觉线索，因此将结果直接转写为文本：

```text
gctf{WH4T_Y0U_D0NT_CH4NG3_Y0U_CH00S3}
```

## 方法总结

PDF 的“不可见”不等于数据不存在。页面渲染只会追踪页面对象树，而 xref 表还可能保存未引用对象、历史增量更新对象或其他任意流。处理这类题时应分开检查可见层、页面引用关系、全部 xref 和流解码后的文件签名。本题新增对象恰好是一个经过 Flate 包装的完整 PNG，提取时使用 `xref_stream()` 而非复制压缩后的 raw stream 即可得到原图。
