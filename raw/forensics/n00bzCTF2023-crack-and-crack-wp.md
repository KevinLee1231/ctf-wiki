# Crack & Crack

## 题目简述

附件是加密 ZIP，内部又包含加密 PDF，必须依次恢复两层口令。仓库中的原始 PDF 只有一页，最终 flag 直接排版在页面中央。

## 解题过程

先用 `rockyou.txt` 对 ZIP 做字典攻击：

```bash
fcrackzip -v -u -D -p /usr/share/wordlists/rockyou.txt flag.zip
```

得到 ZIP 口令 `1337h4x0r`。解压后再破解 PDF：

```bash
pdfcrack -f flag.pdf -w /usr/share/wordlists/rockyou.txt
```

第二层口令为 `noobmaster`。使用该口令打开 PDF 后，我对照原 PDF 的唯一页面逐页观察，页面可见文本与提取结果一致，没有遗漏图形、附注或后续页面。完整内容为：

```text
n00bz{CR4CK3D_4ND_CR4CK3D_1a4d2e5f}
```

## 方法总结

嵌套加密工件要按容器层级逐层处理，并分别记录口令。PDF 解密后仍应核对页数和视觉排版，不能仅凭文本提取工具无输出或少量输出就判断内容完整。
