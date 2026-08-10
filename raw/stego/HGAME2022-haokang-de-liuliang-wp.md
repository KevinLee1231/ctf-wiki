# 好康的流量

## 题目简述

附件是一份邮件通信流量。需要先从 TCP/SMTP 会话中恢复 Base64 编码的图片附件，再对图片做两种 LSB 检查：一个位平面显示条形码形式的前半段，按列抽取 RGB 最低位则得到后半段。

## 解题过程

用 Wireshark 打开流量，定位邮件会话。不同版本对对象导出的识别结果可能不同；若无法直接导出邮件对象，就选择对应 TCP 流并执行“追踪 TCP 流”，从 MIME 正文中找到图片附件的 Base64 内容，去掉邮件头和边界后解码：

```bash
base64 -d attachment.b64 > attachment.png
```

邮件主题和正文含有 LSB 提示。用 StegSolve 打开图片，先浏览各颜色位平面，在 `Green plane 2` 可看到条形码；必要时反相后扫描，得到前半段：

```text
hgame{ez_1mg_
```

然后使用 StegSolve 的 `Analyse -> Data Extract`，设置为：

```text
Bit planes: Red 0, Green 0, Blue 0
Extract By: Column
Bit Order: LSB First
Bit Plane Order: RGB
```

按列提取后，预览区会重复出现 ASCII 文本：

```text
Steg4n0graphy}
```

拼接两部分得到：

```text
hgame{ez_1mg_Steg4n0graphy}
```

原 PDF 的工具截图已全部转写为位平面和提取参数，没有保留只展示按钮位置的界面图片。最终两段文本通过 [HGAME 2022 Week1 流量复现](https://cloud.tencent.com/developer/article/2070184) 交叉核对。

## 方法总结

这道题同时考查网络对象恢复和像素最低位隐写。MIME 附件应先从流量中完整还原，随后分别检查“可视化位平面”和“按位导出数据”两条路径。Data Extract 的行列顺序、位序和通道顺序都会改变输出；只有按列、LSB First、RGB 才能得到可读后半段。
