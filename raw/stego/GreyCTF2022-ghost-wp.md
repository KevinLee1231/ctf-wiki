# GreyCTF2022 - Ghost

## 题目简述

官方 `Solution.pdf` 只有一页，展示十六进制编辑器中的大量 `20` 与 `09` 字节，以及把它们替换为二进制后的结果。隐藏信息由空格和制表符构成，是典型的空白字符隐写。

## 解题过程

逐页视觉核对原 PDF 后确认，决定性映射为：空格字节 `0x20` 记作 0，Tab 字节 `0x09` 记作 1。按文件原顺序提取并每 8 位分组：

```python
data = open('ghost', 'rb').read()
bits = ''.join('0' if b == 0x20 else '1' for b in data if b in (0x20, 0x09))
plain = bytes(int(bits[i:i+8], 2) for i in range(0, len(bits), 8))
print(plain.decode())
```

结果为：

```text
grey{gh0s7_byt3$_n0t_1nvisIbl3}
```

PDF 截图中的十六进制和转换结果均已完整转成文本，没有额外空间关系或视觉标注，因此不保留页面截图。

## 方法总结

看似空白的文本应检查原始字节，而不是依赖编辑器渲染。空格、Tab、换行和不同 Unicode 空白都可承载比特；提取时要保持原始顺序，并根据可打印前缀判断 0/1 映射和每字节位序是否正确。
