# BYUCTF 2023 - Poem

## 题目简述

附件是一段移除了空格和标点的单表替换密文。题目说明明文应是一个短语，因此可结合英文词频和已知前缀 `byuctf` 恢复映射。

## 解题过程

密文为：

```text
epcndkohlxfgvenkzcllkoclivdckskvpkddcyoceipkvrcslkdhycbcscwcsc
```

频率分析给出初始替换，再用 `byuctf`、`the flag is` 等高可信片段校正，最终得到密文字母表映射：

```text
KXVMCNOPHQRDZYIJASLEGWBUFT
```

恢复并重新分词后的全文是：

```text
The flag is byuctf a message so clear a challenge to hackers a line we revere
```

因此 flag 为：

```text
byuctf{a message so clear a challenge to hackers a line we revere}
```

## 方法总结

删除空格会增加分词难度，但不会改变单表替换的字母频率与重复模式。先自动求近似映射，再用题目固定前缀和自然语言结构人工收敛，通常比完全手工试字母稳定。
