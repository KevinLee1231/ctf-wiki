# Tools

## 题目简述

附件是一组层层嵌套的图片与压缩包，目的是串联 F5、Steghide、OutGuess、JPHS 等常见图片隐写工具。每一层压缩包的文件名提示下一步工具，解密密码则写在同层图片的备注中；最终会得到四张二维码碎片。

## 解题过程

第一层提示使用 F5。用 F5 的 Java 提取器处理 `Matryoshka.jpg`：

```powershell
java Extract Matryoshka.jpg -p '!LyJJ9bi&M7E72*JyD'
```

下一层文件名指向 Steghide，使用图片备注给出的密码继续提取：

```powershell
steghide.exe extract -p 'A7SL9nHRJXLh@$EbE8' -sf .\01.jpg
```

随后用 OutGuess 从 `02.jpg` 中恢复嵌入文件：

```powershell
outguess -k "z0GFieYAee%gdf0%lF" -r 02.jpg secret.txt
```

最后一层使用带图形界面的 JPHS，密码为：

```text
rFQmRoT5lze@4X4^@0
```

依次解开全部层级后会得到四张图。它们分别是同一个二维码的左上、右上、左下、右下区域；按照文件名顺序从左到右、从上到下拼接，然后扫码即可得到：

```text
hgame{Taowa_is_N0T_g00d_but_T001s_is_Useful}
```

官方 PDF 中的工具命令、密码和二维码排列关系均已转写为文本。四张中间图只承担二维码碎片的数据作用，不再保留为文档插图。

## 方法总结

面对多层隐写套娃，先记录每一层的载体、提示、密码来源和输出文件，避免工具顺序错位。图片备注、压缩包文件名和扩展名都是题目线索；最终碎片还要根据文件名恢复空间顺序，不能把“成功提取文件”误当成解题终点。
