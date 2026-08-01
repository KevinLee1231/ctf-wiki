# BYUCTF 2023 - Ducky3

## 题目简述

第三个载荷使用出题人随机生成的自定义键盘布局，通用 Ducky decoder 没有对应映射。载荷前半部分依次输入小写字母、大写字母、数字和符号，提供了完整已知明文。

## 解题过程

先按 8 字节切分 `inject.bin`，记录每个报告的 modifier 与 keycode。把前四行与附件 `payload.txt` 中的已知序列对齐：

```text
abcdefghijklmnopqrstuvwxyz
ABCDEFGHIJKLMNOPQRSTUVWXYZ
0123456789
!@#$%^&*()-_
```

由此建立 `(modifier, keycode) -> character` 的自定义映射，再用同一映射解码载荷末尾未知部分。仓库中的 `byuctf.json` 是出题时使用的真实映射，可用于复核，而不是解题必需的先验。

恢复结果：

```text
byuctf{1_h0p3_y0u_enj0yed-thi5_very_muCH}
```

## 方法总结

自定义编码遇到覆盖完整字母表的已知明文时，本质是置换表恢复。先从训练段建表，再对未知段应用；应把 Shift 状态纳入键，而不能只按单字节 keycode 建映射。
