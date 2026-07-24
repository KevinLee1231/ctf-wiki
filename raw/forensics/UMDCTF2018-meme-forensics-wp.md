# UMDCTF 2018 - Meme Forensics

## 题目简述

附件 `FIND_ME.meme` 能被识别为 PNG，但题面提示“改了一个字节”并要求文件雕刻。检查文件尾部可以发现，PNG 后还拼接了一份被破坏魔数的 PDF。

## 解题过程

先定位 PNG 的 `IEND`，再检查后续数据。偏移 `518635` 处出现：

```text
%RDF-1.4
```

标准 PDF 魔数应为 `%PDF-1.4`，因此被修改的是紧随 `%` 的字节：文件偏移 `518636` 的 `R` 应改为 `P`。从 `%` 开始雕刻并修复：

```python
data = bytearray(open("FIND_ME.meme", "rb").read())
start = data.index(b"%RDF-1.4")
data[start + 1] = ord("P")
open("recovered.pdf", "wb").write(data[start:])
```

修复后的 PDF 只有一页。对该页进行视觉核对，页面中的 flag 为：

```text
UMDCTF-{kn0w_ur_m3m3_byt3s}
```

其 SHA-1 与仓库 `README.md` 保存的摘要一致。

## 方法总结

文件雕刻不能只依赖完整魔数。题目已提示单字节损坏时，应同时搜索相近签名，并结合载体文件的逻辑结束位置判断候选。修复后还要用 PDF 解析器和页面渲染双重检查，避免只恢复出一个“看似有头部”但结构不完整的文件。
