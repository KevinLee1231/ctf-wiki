# UMDCTF 2025 - the-mini

## 题目简述

附件 `the mini.puz` 是一个 $5\times5$ 的 Across Lite 填字游戏。文件头表明解答
处于锁定状态，不能直接把 solution 区域当作明文读取。目标是恢复四位数字密钥，
解开被弱混淆的答案；决定性障碍是可逆变换和小密钥空间，因此归入 `crypto`。

## 解题过程

`.puz` 文件包含魔数 `ACROSS&DOWN`、宽高、线索数、solution 状态和一个 16 位
scrambled checksum。本题解析出的关键字段为：

```text
width = 5
height = 5
solution_state = locked
scrambled_checksum = 0x6a7f
```

锁定算法可从 [puzpy 的公开实现](https://github.com/alexdej/puzpy) 完整确认。
正文所需的算法如下：

1. 把行优先网格转置为列优先字符串，并暂时移除黑格 `.`；
2. 将四位密钥拆成四个数字；
3. 对每个密钥数字，先按四位数字循环对字母做 Caesar 位移，再按当前数字循环切分，
   最后把字符串做一次奇偶位置交错洗牌；
4. 解锁时逆序执行反洗牌、反向循环切分和反位移；
5. 把黑格恢复到原位置，再转回行优先网格。

密钥仅在 `1000` 到 `9999` 之间，可以全部枚举。每个候选解答按列优先顺序去掉黑格，
再计算文件格式使用的 16 位校验：

```python
def data_cksum(data):
    cksum = 0
    for b in data:
        lowbit = cksum & 1
        cksum >>= 1
        if lowbit:
            cksum |= 0x8000
        cksum = (cksum + b) & 0xffff
    return cksum
```

唯一通过 `0x6a7f` 校验的密钥为：

```text
5727
```

逆变换后的完整 $5\times5$ solution 字符串是：

```text
UMDCTFCANYOUBEATMYTIME...
```

末尾三个点是黑格。结合题目要求的 flag 格式，答案为：

```text
UMDCTF{CANYOUBEATMYTIME}
```

## 方法总结

Across Lite 的“锁定”是四位密钥驱动的可逆字符置换，并不提供密码学安全性。
分析这类格式时应同时读取状态位和校验字段：前者说明答案经过变换，后者为枚举密钥
提供可靠判据。正文保留了必要的变换顺序，因此即使不访问外链，也能复现恢复过程。
