# Greycademy2025 filefactory

## 题目简述

附件名为 `flag.pdf`，但常规 PDF 阅读器无法打开。题目要求识别文件的真实格式，逐层修复被篡改的魔数，最终恢复其中的 flag。

## 解题过程

先查看文件头而不是相信扩展名：

```bash
xxd -l 32 flag.pdf
file flag.pdf
7z l flag.pdf
```

开头是 `50 4b 03 04`，即 ZIP 本地文件头。`7z l` 也能列出其中唯一的 `flag.png`，所以直接解包：

```bash
7z x flag.pdf
```

内层文件仍无法作为 PNG 打开。其前 16 字节如下：

```text
4a 45 53 53 0d 0a 1a 0a 00 00 00 0d 49 48 44 52
J  E  S  S                         I  H  D  R
```

标准 PNG 签名应是 `89 50 4e 47 0d 0a 1a 0a`，这里只有前四字节被替换成了 `JESS`，后面的换行标记和 `IHDR` 都仍然正确。把完整八字节签名写回即可避免误改后续结构：

```python
from pathlib import Path

path = Path("flag.png")
data = bytearray(path.read_bytes())
data[:8] = b"\x89PNG\r\n\x1a\n"
Path("repaired.png").write_bytes(data)
```

修复后的 1200 × 630 图片只写有一行手写文本，转写为：

```text
grey{these_files_are_kinda_weird_but_im_weirder}
```

## 方法总结

文件扩展名只是提示，不是格式证据。可靠流程是从魔数、容器目录和内部结构逐层验证：外层伪装 PDF 实为 ZIP，内层又是签名损坏的 PNG。修复时只改已被证据锁定的字节，并在每一层用 `file`、解包器或 PNG 校验器复核。最终图片只有文字而没有额外视觉信息，因此正文转写即可，无需归档一张代码或 flag 截图。
