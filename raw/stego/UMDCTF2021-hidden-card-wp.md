# Hidden Card

## 题目简述

题目给出一张牌堆图片。常见的元数据、附加文件和简单 LSB 检查没有直接结果，实际使用的是 DIIT 工具中的 BattleSteg 算法，将一个文本文件分散嵌入图像像素。

![看似普通的扑克牌牌堆图片，BattleSteg 算法在像素选择路径中嵌入了文本](UMDCTF2021-hidden-card-wp/battle-steg-card-stack.png)

## 解题过程

使用 Digital Invisible Ink Toolkit（DIIT）打开原始 PNG，选择 BattleSteg 解码器。BattleSteg 不按线性像素顺序读取，而是根据“战舰”式区域选择策略分散写入数据，所以普通的逐通道 LSB 提取不会自然得到明文。

在 DIIT 中执行：

```text
Decode -> BattleSteg
Input image: DeckOfCards.png
Output file: battle.txt
```

本题不需要额外口令。解码生成的 `battle.txt` 中包含：

```text
UMDCTF-{c4rd_1n_a_c4rd$t4ck}
```

## 方法总结

图像隐写载体没有明显异常时，应根据题名、附件来源和像素统计考虑具体工具/算法。BattleSteg 的像素遍历策略与通用 LSB 不同，算法不匹配时即使位平面里确有数据也无法正确重组。保留原 PNG，避免重新编码破坏最低位。
