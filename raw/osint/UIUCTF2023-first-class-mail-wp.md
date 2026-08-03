# UIUCTF 2023 First class mail Writeup

## 题目简述

题目给出一张 Jonah 桌面的照片，要求确定其所在地邮政编码。画面中有书、文具、扑克牌、耳机、保险公司信件等物品；真正的关键位于 AirPods 盒下方露出的黑色长短竖条。

![桌面照片中 AirPods 盒下方露出一段长短相间的 POSTNET 条码，旁边的保险公司信件是干扰项](./UIUCTF2023-first-class-mail-wp/postnet-clue-tabletop.jpg)

提示 “code is cool” 指向一种编码，而 Erie Insurance 和 Lanyi Insurance 的地址信息只是诱导地理搜索的干扰项。

## 解题过程

照片中的竖条符合 USPS POSTNET：首尾各有一根长保护条，中间每 5 根为一组；每组恰有 2 根长条，并按长短模式表示一个十进制数字。用 `T` 表示长条、`S` 表示短条，映射为：

```text
0 TTSSS    1 SSSTT    2 SSTST    3 SSTTS    4 STSST
5 STSTS    6 STTSS    7 TSSST    8 TSSTS    9 TSTSS
```

忽略首尾保护条，对照片中的 50 根数据条按五根分组，得到：

```text
STTSS  TTSSS  STTSS  STTSS  SSSTT
SSSTT  SSSTT  SSTST  SSTTS  STSST
```

按映射解码为：

```text
6066111234
```

仓库中的解码示意图也显示了相同的高低条序列和数字结果：

![POSTNET 条码逐组解码为 6066111234，最后一位 4 是校验数字](./UIUCTF2023-first-class-mail-wp/decoded-postnet-barcode.png)

POSTNET 的 10 个数字组表示 9 位 ZIP+4 数据加一位校验数字。前 9 位为 `606611123`，即 `60661-1123`；各位和为 26，所以末位校验数字为 4，使总和成为 30。题目只要求五位 ZIP Code，因此取前五位：

```text
uiuctf{60661}
```

## 方法总结

照片 OSINT 不应默认从醒目的品牌或地址入手。先盘点所有结构化符号，再结合提示识别编码制式。本题的 POSTNET 还可用校验位自证：数据位之和加校验位应为 10 的倍数。保留原图与解码条带，并在正文中写出分组、码表和校验过程，可以让结论不依赖视觉猜测。
