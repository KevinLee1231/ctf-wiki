# week2Calender

## 题目简述

附件使用“日历密码”表示一串字母。题目名虽写作 `Calender`，实际考点是把日历中的日期数字按 $1\leftrightarrow a,…,26\leftrightarrow z$ 映射为文本。

## 解题过程

依次读取题图中的日期数字，并使用

$$
1\to a,\quad2\to b,\quad\ldots,\quad26\to z
$$

进行替换。按题目分组将单词间隔写成下划线，得到：

```text
welcome_to_the_second_week
```

仓库中的 `flag.txt` 与该结果一致，最终提交：

```text
0xGame{welcome_to_the_second_week}
```

## 方法总结

日历密码本质是 $1$ 到 $26$ 的字母序号替换。需要额外注意题目要求的大小写、单词分隔符和 flag 包装格式，不要把题名中的拼写错误带入答案。
