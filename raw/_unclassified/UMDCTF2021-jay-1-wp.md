# Jay 1

## 题目简述

题目要求在一维整数序列中求连续子数组的最大和。输入规模不适合枚举所有起止位置，标准解法是 Kadane 算法，时间复杂度 $O(n)$。

## 解题过程

令 `ending` 表示“必须以当前位置结尾”的最大子数组和，`best` 表示截至当前位置的全局最大值：

$$
\text{ending}_i=\max(a_i,\text{ending}_{i-1}+a_i),
$$

$$
\text{best}_i=\max(\text{best}_{i-1},\text{ending}_i).
$$

实现时用首元素初始化，才能正确处理全负数组：

```python
def max_subarray(values):
    ending = best = values[0]
    for value in values[1:]:
        ending = max(value, ending + value)
        best = max(best, ending)
    return best
```

按服务协议读取每组数据并提交结果，完成所有测试后返回：

```text
UMDCTF-{K4d4n35_41g0r1thm}
```

## 方法总结

Kadane 算法把二维的区间选择压缩成一个局部状态：前缀贡献为负时从当前元素重新开始。不要把状态初始化为 0，否则全负输入会错误返回空子数组的和。
