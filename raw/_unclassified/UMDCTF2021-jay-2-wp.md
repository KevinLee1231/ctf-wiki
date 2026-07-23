# Jay 2

## 题目简述

题目把上一关扩展到二维矩阵，要求求最大和子矩形。直接枚举四条边并逐格求和代价过高，可以固定上下边界，把二维问题压缩成一维最大子数组。

## 解题过程

枚举上边界 `top`，维护每列在 `top..bottom` 之间的累加和 `compressed`。每增加一行作为 `bottom`，就在该一维数组上运行 Kadane：

```python
def max_rectangle(matrix):
    rows, cols = len(matrix), len(matrix[0])
    answer = matrix[0][0]

    for top in range(rows):
        compressed = [0] * cols
        for bottom in range(top, rows):
            for col in range(cols):
                compressed[col] += matrix[bottom][col]

            ending = best = compressed[0]
            for value in compressed[1:]:
                ending = max(value, ending + value)
                best = max(best, ending)
            answer = max(answer, best)

    return answer
```

复杂度为 $O(R^2C)$；若列数远小于行数，可先转置以枚举较短维度。通过服务全部测试后得到：

```text
UMDCTF-{Pr0f3ss0r_Gr3n4nd3r}
```

## 方法总结

最大和子矩形的核心是“固定两条边，压缩另一维”。列累加避免重复计算矩形内部和，Kadane 负责选择最佳左右边界。初始化仍需兼容全负矩阵，并注意输入行列方向。
