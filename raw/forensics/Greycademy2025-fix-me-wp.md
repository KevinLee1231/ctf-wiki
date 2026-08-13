# Fix Me!

## 题目简述

附件名为 PNG，但文件识别工具只报告 `data`，说明扩展名与真实文件头不一致。首 8 字节是 ASCII `GREYHATS`，而后面紧跟合法的 `IHDR` 块；核心任务是修复被覆盖的 PNG 文件签名。

## 解题过程

原始开头如下：

```text
47 52 45 59 48 41 54 53 00 00 00 0D 49 48 44 52
G  R  E  Y  H  A  T  S              I  H  D  R
```

PNG 的固定签名应为 `89 50 4E 47 0D 0A 1A 0A`。可以用十六进制编辑器覆盖前 8 字节，也可以使用下面的等价脚本生成修复副本：

```python
from pathlib import Path

data = bytearray(Path("fixme.png").read_bytes())
data[:8] = bytes.fromhex("89 50 4e 47 0d 0a 1a 0a")
Path("fixme-repaired.png").write_bytes(data)
```

修复后文件被识别为 1536×864、RGBA 的标准 PNG，打开后可读出手写内容：

```text
grey{sign4tur3s_c001}
```

修复图只承载这行可转写文本，没有额外视觉证据，因此不在归档中重复保存图片。

## 方法总结

当图片无法打开时，应先核对 magic bytes，而不是先怀疑编码器。若关键结构块仍在、只有固定签名被替换，最小修复是只改签名字节，避免重编码破坏原始数据。PNG 的 8 字节签名是此类题最直接的识别信号。
