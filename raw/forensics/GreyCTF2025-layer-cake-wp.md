# Layer Cake

## 题目简述

题目把附件命名为 `layer cake.mp3`，但文件实际是多层伪装的 Office Open XML 文档。需要通过文件头和内部结构识别真实格式，再从 DOCX 的组成文件中定位隐藏注释。

## 解题过程

附件开头四字节是看似合理的 MP3 帧同步头：

```text
FF FB 68 30
```

但紧随其后的字段出现目录名 `layers/`，文件内部还存在多个 `50 4B 03 04` ZIP 本地文件头。将开头四字节修复为 ZIP 签名并改名为 `layer-cake.docx`：

```python
from pathlib import Path

data = bytearray(Path("layer cake.mp3").read_bytes())
data[:4] = b"PK\x03\x04"
Path("layer-cake.docx").write_bytes(data)
```

DOCX 本质上是 ZIP 容器，解包后可见 `docProps`、`word`、`[Content_Types].xml` 等标准组成。题目把全部内容再套在顶层 `layers/` 目录中；直接搜索 flag 前缀即可：

```bash
unzip -q layer-cake.docx -d extracted
grep -Rho 'grey{[^<]*}' extracted
```

命中位置是 `layers/word/styles.xml` 中的一条 XML 注释，内容为：

```text
grey{s0_f3w_lay3r5_w00p5}
```

部分解压工具会扫描并容忍原文件中较后的 ZIP 签名，所以未经修复也可能列出成员；修复文件头仍能明确说明题目设计的格式伪装层。

## 方法总结

扩展名和最前面的魔数都可能被伪造，识别格式时还要结合内部结构、可打印路径和后续签名。DOCX 不只是正文 `document.xml`，还包含样式、关系、属性等多个 XML 文件；对这类题应解包后全局检索，而不是只在办公软件中查看可见页面。
