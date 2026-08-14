# filefactory

## 题目简述

附件名为 `flag.pdf`，但无法作为正常 PDF 打开。目标是识别它的真实容器格式，取出内部图片并修复被篡改的文件头，最终读取 flag。

## 解题过程

文件扩展名不能代表实际格式。使用 `file` 检测附件会得到：

```text
flag.pdf: Zip archive data, at least v2.0 to extract
```

其起始字节也是 ZIP 的签名 `50 4b 03 04`，归档中包含 `flag.png`。解压后再次检查图片，工具只会报告 `data`。十六进制开头如下：

```text
4a 45 53 53 0d 0a 1a 0a 00 00 00 0d 49 48 44 52
 J  E  S  S                         I  H  D  R
```

`IHDR` 与后续结构表明它仍是 PNG，只是前四字节被改成了 ASCII `JESS`。标准 PNG 的八字节签名为：

```text
89 50 4e 47 0d 0a 1a 0a
```

用十六进制编辑器把文件开头的 `4a 45 53 53` 改为 `89 50 4e 47`，保存后即可正常打开图片。图片中的文字为：

```text
grey{these_files_are_kinda_weird_but_im_weirder}
```

## 方法总结

该题包含两层格式伪装：PDF 扩展名下实际是 ZIP，ZIP 中的 PNG 又被破坏了魔数。排查异常附件时，应按“检测真实类型—检查容器—核对内部文件签名”的顺序处理，而不能依赖扩展名。
