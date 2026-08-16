# HackINI2024 eQual5bUtNoTeQual5

## 题目简述

服务端把用户输入转换为 Python 浮点数，并检查变量是否等于自身。普通数值总满足 `a == a`，程序会拒绝；只有绕过这个判断才能读取 flag。

## 解题过程

核心逻辑等价于：

```python
a = float(input())
if a == a:
    print("Nope")
else:
    print(flag)
```

IEEE 754 定义的 NaN（Not a Number）具有特殊比较语义：它不等于包括自身在内的任何值。Python 的 `float()` 能直接解析字符串 `nan`，因此向服务提交：

```text
nan
```

此时 `a == a` 为 `False`，程序进入输出 flag 的分支：

```text
shellmates{eKwaL5_8U7_N07_eKWAl5_W17h_Nan}
```

## 方法总结

当程序把字符串转换为浮点数后再进行自反性、大小关系或范围检查时，应考虑 `nan`、`inf` 和 `-inf` 等特殊值。NaN 破坏了“任何值都等于自身”这一对普通数成立的直觉，是本题最小且直接的绕过方式。防御时应显式使用 `math.isfinite()` 或 `math.isnan()` 拒绝特殊浮点值。
