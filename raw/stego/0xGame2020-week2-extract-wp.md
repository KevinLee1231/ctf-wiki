# week2extract

## 题目简述

附件 `flagg.jpg` 表面是 `440 × 350` 的 JFIF 图片，但 JPEG 结束位置之后追加了一个 ZIP。压缩包内的 `flag.png` 是二维码，二维码内容不是 flag，而是下一步工具提示 `stegpy`；真正的 flag 仍隐藏在该 PNG 中。

## 解题过程

用 `binwalk` 检查文件结构：

```bash
binwalk flagg.jpg
```

关键结果为：

```text
DECIMAL  HEXADECIMAL  DESCRIPTION
0        0x0          JPEG image data, JFIF standard 1.01
44213    0xACB5       Zip archive data, name: flag.png
45715    0xB293       End of Zip archive
```

由于 ZIP 允许前置任意数据，`unzip` 可以直接忽略前面的 JPEG 数据并提取文件：

```bash
unzip flagg.jpg flag.png
```

扫描 `flag.png` 中的二维码，得到文本：

```text
stegpy
```

这表示 PNG 还是一个由 stegpy 生成的隐写容器。对提取出的原始 PNG 执行：

```bash
stegpy flag.png
```

stegpy 会读取其嵌入头和像素最低位中的载荷，并输出隐藏的 flag。不要对 `flag.png` 进行截图、转码或重新保存，否则最低位数据可能被破坏。

## 方法总结

- 核心技巧：先从 JPEG 尾部 carving 出 ZIP，再按二维码提示进入第二层 PNG 隐写。
- 识别信号：`binwalk` 在 JPEG 正常数据之后报告 ZIP，提取出的二维码只给出工具名而非 flag。
- 复用要点：每一层输出都决定下一步工具；隐写载体必须保持原始字节，避免图像软件重编码。
