# least and most

## 题目简述

题名对应 least significant bit 与 most significant bit。PNG 的 RGB 三个通道中同时藏有两段文本：一段位于每个颜色字节的最低位（bit 0），另一段位于最高位（bit 7），需要分别提取后按语义拼接。

## 解题过程

在 StegSolve 的 Data Extract 中依次完成两次提取：

1. 勾选 Red、Green、Blue，选择 bit plane 0，以行优先、RGB 顺序读取最低有效位；
2. 保持相同通道与遍历顺序，改选 bit plane 7，读取最高有效位。

最低位预览的有效十六进制字节以 `30 78 47 61 6d 65 7b 6c 73 62 5f 63 6f 6d` 开头，对应文本：

```text
0xGame{lsb_com
```

最高位预览以 `62 69 6e 65 64 5f 77 69 74 68 5f 6d 73 62 7d` 开头，对应后半段：

```text
bined_with_msb}
```

按前后语义连接，完整 flag 为：

```text
0xGame{lsb_combined_with_msb}
```

若结果乱码，应首先确认通道顺序、位编号和像素遍历方向与上述设置一致。

## 方法总结

最低位隐写利用对视觉影响很小的 bit 0，最高位则把信息放在对数值贡献最大的 bit 7；同一图像可以同时承载两条位流。提取时必须记录通道、位平面、颜色顺序和遍历方向，并用 flag 结构验证片段拼接顺序。
