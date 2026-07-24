# The Lost Flag

## 题目简述

附件是一份两页 PDF。逐页视觉核对后，第一页只是标题页，第二页是 $\sqrt{3}$ 为无理数的反证法证明，页面上没有完整 flag；第二页末尾孤立的字符 `c` 则提示文件可能存在被正常交叉引用表忽略的增量内容。README 的 `?????-85` 进一步指向 ASCII85。

## 解题过程

检查 PDF 尾部可见，活动的：

```text
startxref
215923
%%EOF
```

指向对象 68 所在的交叉引用表。但文件后面还存在对象 69，另一个未被活动 `startxref` 采用的 xref 位于偏移 `216352`。对象 69 声明：

```text
/Length 51
/Filter /FlateDecode
```

因此不必依赖阅读器渲染，直接从原始文件中定位该 stream，先做 zlib 解压，再按提示做 ASCII85 解码：

```python
import base64
import re
import zlib
from pathlib import Path

data = Path("The_Lost_Flag.pdf").read_bytes()
match = re.search(rb"69 0 obj.*?stream\r?\n", data, re.S)
assert match is not None

compressed = data[match.end():match.end() + 51]
ascii85 = zlib.decompress(compressed)
flag = base64.a85decode(ascii85, adobe=False)
print(flag.decode())
```

中间层为：

```text
<D>kK<(8Hd7VQmaF@0"s?WSt(BO=5L1G`B7E\K1Z5CE
```

最终得到：

```text
UMDCTF-{FirstCTF_W1th_Fr33_Pr00f?}
```

其 SHA-256 与 README 中的 `88e68d9b3459f782eb66abf33de49fb36171e67af818579792d0784e7f947062` 一致。

## 方法总结

PDF 的可见页面只是解析器沿当前 xref 构建出的视图，未被引用的对象仍可能留在文件尾。取证时应同时检查页面渲染、`startxref`、增量更新和孤立对象。本题的完整链条是“未激活对象 69 → FlateDecode → ASCII85”，而不是从证明正文中猜 flag。
