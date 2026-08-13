# Greycademy2025 hELLO KItty

## 题目简述

附件 `hellokitty.jpg` 表面是一张普通图片，题面明确提示使用常见隐写工具。目标是识别 JPEG 中的 steghide 载荷并提取其中的文本。

## 解题过程

先让 `steghide` 查看载体信息。题目没有提供口令，而载荷使用空口令，因此直接提取即可：

```bash
steghide info hellokitty.jpg
steghide extract \
  -sf hellokitty.jpg \
  -p "" \
  -xf answer.txt
cat answer.txt
```

实际提取结果为：

```text
its not so easy afterall.. its only the begining of steganography!

grey{easy_$T3g0_wIth_K1TTy}
```

最终 flag：

```text
grey{easy_$T3g0_wIth_K1TTy}
```

仓库同时保留了原始 `secret` 和官方命令说明，内容与本地 `steghide` 提取结果一致。

## 方法总结

这是一道 steghide 入门题。JPEG 能正常显示并不意味着没有嵌入数据；当题面已给出 stego 工具线索时，应先验证常见容器及空口令，再考虑字典攻击。载体画面本身不参与定位或重组，真正证据是可复现的提取命令与输出，所以无需把普通猫图复制进 WP。
