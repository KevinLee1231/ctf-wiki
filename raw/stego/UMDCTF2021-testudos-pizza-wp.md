# Testudo's Pizza

## 题目简述

题目给出一张能够正常打开的 JPEG。图片画面只是掩护，文件尾追加了富文本片段，flag 以普通字符串形式藏在 JPEG 结束标记之后。

## 解题过程

先确认图片格式和文件尾：

```bash
file hiddenmsg.jpg
strings -a hiddenmsg.jpg | tail -n 40
```

输出中可以看到 RTF 控制字和智能引号，说明 JPEG 后面还附有文本对象。直接限定 flag 前缀搜索：

```bash
strings -a hiddenmsg.jpg | grep 'UMDCTF'
```

得到：

```text
UMDCTF-{W3_ar3_th3_b3st_P1ZZ3r1a}
```

也可定位 JPEG 的 `FF D9` 结束标记，将其后的字节单独保存，再按 RTF 文本解析。

## 方法总结

文件能正常显示不代表结束标记后没有数据。JPEG 解码器通常忽略 `FFD9` 后的尾随内容，适合做简单附加隐写。先用 `strings` 和十六进制结构确认，再决定是否需要 `binwalk` 或手工切割；本题无需保留没有额外视觉价值的封面截图。
