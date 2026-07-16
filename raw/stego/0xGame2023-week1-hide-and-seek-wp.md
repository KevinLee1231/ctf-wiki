# hide and seek

## 题目简述

JPEG 中包含由 Steghide 嵌入的文件，隐写口令强度很低。StegSeek 可以利用 Steghide 的已知格式快速测试字典口令，并在命中后直接提取载荷。

## 解题过程

使用常见口令字典 `rockyou.txt` 对附件执行：

```bash
stegseek "hide and seek.jpg" rockyou.txt
```

工具命中口令 `071sbrmw`。输出同时表明载荷的原始文件名是 `flag.txt`，默认实际写出的文件名则是 `hide and seek.jpg.out`：

```text
Found passphrase: "071sbrmw"
Original filename: "flag.txt"
Extracting to "hide and seek.jpg.out"
```

随后读取提取文件：

```bash
cat "hide and seek.jpg.out"
```

得到：

```text
0xGame{Just_hide_and_S33k}
```

若希望分开验证，可先用 `stegseek --crack` 找出口令，再执行 `steghide extract -sf "hide and seek.jpg" -p 071sbrmw`。

## 方法总结

对 JPEG/BMP/WAV/AU 中疑似 Steghide 载荷，可先查看元数据与提示，再在授权的题目环境中用字典测试弱口令。WP 应记录工具、字典、命中的具体口令和提取文件名，避免只保留一张终端截图而无法复现。
