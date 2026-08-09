# Random

## 题目简述

服务用已弃用的 C++ `random_shuffle` 打乱字符串。新进程没有调用 `srand`，因此每次启动都从同一全局伪随机状态开始，第一次置换完全可预测。

## 解题过程

输入必须至少 10 字符且不能重复。先向一个新进程提交探针：

```text
0123456789
```

程序在进入排序函数前调用一次 `random_shuffle`，随后立即打印置换结果。因为探针字符唯一，可以从输出恢复该次置换的逆映射：

```python
probe = "0123456789"
scrambled = first_line_from_fresh_process(probe)

answer = [""] * len(probe)
for output_pos, ch in enumerate(scrambled):
    answer[int(ch)] = probe[output_pos]
answer = "".join(answer)
```

再连接一个同样从默认随机状态启动的新进程，提交 `answer`。相同置换会把它变成已排序的 `0123456789`，排序函数第一次检查就成功并返回：

```text
n00bz{5up3r_dup3r_ultr4_54f3_p455w0rd}
```

## 方法总结

伪随机置换的安全性取决于不可预测的状态。固定默认种子使跨进程第一次输出可复现；使用唯一字符探针即可恢复排列并构造逆置换输入。
