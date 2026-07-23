# Not Slick

## 题目简述

题目给出一个无法正常识别的文件。检查文件头和文件尾后可以发现，PNG 的全部字节被整体倒序保存，因此正常魔数出现在文件末端。

## 解题过程

用十六进制查看器检查两端。正常 PNG 开头应为：

```text
89 50 4e 47 0d 0a 1a 0a
```

而附件的字节顺序与之相反。将整个文件按字节逆序写出：

```python
from pathlib import Path

data = Path("notslick").read_bytes()
Path("recovered.png").write_bytes(data[::-1])
```

恢复后的文件通过 `file`/PNG 校验并能正常打开，图中给出：

```text
UMDCTF-{abs01ute1y_r3v3r53d}
```

## 方法总结

文件损坏题先检查魔数、尾标记和块结构。本题不是按位翻转，也不是逐块倒序，而是整个字节数组反转；`data[::-1]` 精确复现这一变换。恢复后应再用格式解析器验证，而不是仅凭图片查看器“能打开”就结束。
