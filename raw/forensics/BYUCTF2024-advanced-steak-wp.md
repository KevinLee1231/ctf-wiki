# Advanced Steak

## 题目简述

附件是一份被随机数据部分覆盖的 NTFS 原始镜像。文件系统元数据已损坏，目标 `.cow` 又是自定义格式，常规文件系统解析和默认文件签名库都无法直接恢复它。题目另给两个样本与解密器，用于推断签名并手工雕刻文件。

## 解题过程

先比较两个已知 `.cow` 样本的首尾字节：

```text
文件头：13 37 be ef
文件尾：4d 6f 6f 6f    # ASCII "Mooo"
```

在 `Mad Cow.001` 中搜索头标记，再从该位置向后搜索尾标记，连同尾部四字节一并导出。可用十六进制编辑器操作，也可先记录偏移后切片：

```python
data = open("Mad Cow.001", "rb").read()
start = data.find(bytes.fromhex("1337beef"))
end = data.find(bytes.fromhex("4d6f6f6f"), start) + 4
assert start >= 0 and end >= start + 4
open("carved.cow", "wb").write(data[start:end])
```

`cow_decryptor.py` 揭示转换规则：所有字节与 `0xff` 异或，再把头四字节修复为 PNG 签名 `89504e47`，把末四字节修复为 PNG 尾部 `ae426082`。运行：

```text
python3 cow_decryptor.py -i carved.cow -o recovered.png
```

恢复图像中的文字为：

```text
byuctf{incredi-bull}
```

原仓库中的牛图与 flag 图只承载可直接转写的文本，没有保留为 WP 插图。

## 方法总结

文件系统损坏不等于文件内容消失。先从已知样本提取稳定的首尾签名，再在原始镜像中做字节级雕刻，最后按自定义封装规则恢复标准格式。输出前应同时验证头、尾与文件长度，避免把邻接扇区误带入结果。
