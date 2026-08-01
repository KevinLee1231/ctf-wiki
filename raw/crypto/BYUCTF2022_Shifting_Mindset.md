# BYUCTF 2022 - Shifting Mindset

## 题目简述

题面强调 “American computer” 和 “shifting”，附件则只包含数字行上方的符号，例如 `(`、`@`、`!`。这些符号来自美式键盘按住 Shift 后输入数字键的结果。

## 解题过程

先对每个符号执行“取消 Shift”映射：

```text
! @ # $ % ^ & * ( )
1 2 3 4 5 6 7 8 9 0
```

保留原有空格分组后，密文变成一组十进制数：

```text
9 20 8 9 14 11 13 25 19 8 9 6 20 11 5 25 9 19 19 20 21 3 11
```

再用 A1Z26，即 `1 -> A`、`26 -> Z`，可读出：

```text
I THINK MY SHIFT KEY IS STUCK
```

去掉空格并套用题目格式得到主答案；官方同时接受带单词下划线的等价写法：

```text
byuctf{ithinkmyshiftkeyisstuck}
```

## 方法总结

本题是两层表示转换：先由键盘布局恢复数字，再由 A1Z26 恢复字母。题面中的 “American” 很重要，因为不同键盘布局的 Shift 符号并不完全相同。
