# UMDCTF 2019 - ZTC матрёшка

## 题目简述

题名意为“套娃”。附件从磁盘镜像开始，依次嵌套 PCAP、DOCX、JPEG 尾随 ZIP、7z、SQLite、plist 和 ZIP；每一层都需要依据真实格式继续取证。题目还混入了自包含 ZIP、图片和压缩炸弹等干扰项。

## 解题过程

磁盘的第一个分区从第 1 扇区开始，是 FAT12。The Sleuth Kit 能看到已删除的 `capture.pcap`：

```bash
mmls matryoshka.img
fls -r -o 1 matryoshka.img
icat -o 1 matryoshka.img 62598 > capture.pcap
```

PCAP 中只有一条关键 HTTP 会话，请求路径是 `/матрёшка.docx`。导出 HTTP 对象即可还原文档：

```bash
tshark -r capture.pcap --export-objects http,http-objects
file http-objects/*
```

DOCX 本质上是 ZIP。文档提示不要相信扩展名、文件头和归档表面结构。解包后检查 `word/media/image2.jpg`，JPEG 的图像结束标记后还附加了一个 ZIP：

```bash
unzip matryoshka.docx -d docx
binwalk docx/word/media/image2.jpg
dd if=docx/word/media/image2.jpg of=appended.zip bs=1 skip=715360
unzip -l appended.zip
```

归档内有 `droste`、`pigs`、`doll`、`dog`、`dream` 等对象。`droste` 是字节完全相同的自包含 ZIP，图片和梦境文件也是干扰；真正的下一层是 `doll`。它经 zlib 解压后成为一个 7z：

```python
import zlib

open("doll.7z", "wb").write(zlib.decompress(open("doll", "rb").read()))
```

这里必须先运行 `7z l doll.7z`，不能直接全部解压。清单中有七个逻辑大小为 64 GiB 的条目，是压缩炸弹；只选择性提取很小的 `08s.data`：

```bash
7z e doll.7z 08s.data
sqlite3 08s.data '.tables'
sqlite3 08s.data 'select name, length(data) from dolls;'
```

数据库表 `dolls` 中的 `babushka1` 是二进制 plist。解析后得到一个 ZIP 字节串，其中有两个同名 `DATAFILE` 条目，必须按条目序号分别读取，不能只按文件名覆盖提取：

```python
import io
import plistlib
import sqlite3
import zipfile

db = sqlite3.connect("08s.data")
blob = db.execute(
    "select data from dolls where name='babushka1'"
).fetchone()[0]
archive_bytes = plistlib.loads(blob)["archive"]

with zipfile.ZipFile(io.BytesIO(archive_bytes)) as archive:
    for index, info in enumerate(archive.infolist()):
        print(index, info.filename, archive.read(info))
```

第一份 `DATAFILE` 的俄文说明要求把 `УМДЦТФ-Самиздат` 转写为拉丁字母，并提示第四个字母应为 `C`。最终得到：

```text
UMDCTF-Samizdat
```

其 SHA-256 为 `daf6586932782cb59859e51a4ab000dc97c83f73ad6613e08e28863833340373`，与官方摘要一致。

## 方法总结

套娃题的核心不是无条件递归解压，而是每一层都重新做格式识别、结构检查和大小评估。遇到重复文件名要按归档条目读取；遇到极高压缩比要先列清单并选择性提取。路径名、扩展名和首个看似合理的对象都可能是诱饵，最终结论应由格式结构和官方摘要共同验证。
