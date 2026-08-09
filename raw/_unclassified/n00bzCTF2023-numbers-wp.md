# Numbers

## 题目简述

服务在 60 秒内连续提问 100 次：数字 $x$ 在从 1 到 $y-1$ 的十进制表示中一共出现多少次。主障碍是快速、准确地解析并自动回答，属于通用编程题。

## 解题过程

源码实际使用 `range(1, y)`，所以不包含上界 $y$。直接拼接或逐项计数已足以应对本题范围：

```python
def count_digit(x, y):
    digit = str(x)
    return sum(str(value).count(digit) for value in range(1, y))
```

脚本循环读取 `How many X's appear till Y?`，提取 `X` 和 `Y`，调用上述函数并发送结果。连续答对 100 轮后得到：

```text
n00bz{4n_345y_pr0gr4mm1ng_ch4ll}
```

## 方法总结

这类交互题最容易错在区间边界和题面解析。应以服务端实现为准确认上界是否包含，再自动化 100 轮交互；本题数据规模无需复杂数位 DP。
