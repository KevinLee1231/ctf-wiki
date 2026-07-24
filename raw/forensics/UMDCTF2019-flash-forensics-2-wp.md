# UMDCTF 2019 - Flash Forensics II

## 题目简述

第二题仍提供 `ubuntu_live.img`，并说明曾用一份 PDF 替换 Ubuntu 文件。与第一题的基准镜像相比，新镜像多出一段数据，但 PDF 文件头被轻微破坏。

## 解题过程

先比较两份镜像的长度和差异区域：

```bash
cmp -l flash1.img flash2.img | head
stat -c '%n %s' flash1.img flash2.img
```

第二份镜像多出 `20502` 字节。搜索 PDF 版本标记时，在偏移 `10035686` 处看到：

```text
%PDG-1.4
```

标准 PDF 头应为 `%PDF-1.4`，其中 `F` 被改成了 `G`。从该偏移开始 carve 数据，并修复一个字节：

```python
from pathlib import Path

image = Path("ubuntu_live.img").read_bytes()
start = image.index(b"%PDG-1.4")
end = image.index(b"%%EOF", start) + len(b"%%EOF")

pdf = bytearray(image[start:end])
pdf[:8] = b"%PDF-1.4"
Path("recovered.pdf").write_bytes(pdf)
```

修复后的单页 PDF 可以正常渲染，页面给出的 flag 为：

```text
UMDCTF-{w4lk_th3_b1ns}
```

其 SHA-256 为 `933f229902cc135176a39571e69d6e0f7c18232eba7b23e2e816a427f39deec6`，与官方摘要一致。

## 方法总结

有上一题镜像作为基准时，差分比盲目扫描更有效。文件签名只损坏一个字符并不会抹掉主体内容；应结合长度差异、近似 magic、尾标记和实际渲染结果完成恢复，而不是只依赖严格的文件识别工具。
