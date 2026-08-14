# Distinct

## 题目简述

WelcomeCTF2021 的 Distinct 读取 16 个 64 位整数，排序后根据是否重复调用函数指针 `handler`。全局数组 `nums[16]` 后紧邻 `handler`，排序函数把长度 16 当作最后下标，产生一次越界访问。目标是让最终调用转向 `win()`。

## 解题过程

漏洞位于两个循环的边界：

```c
for (int i = 0; i <= len; i++) {
    for (int j = i; j <= len; j++) {
        /* sort */
    }
}
```

调用时 `len` 为 16，合法下标却只有 $0\ldots15$。因此 `arr[16]` 实际就是相邻的函数指针 `handler`，排序会把它当作第 17 个整数移动。

PIE/ASLR 使 `win` 的绝对地址变化，但 `unique` 与 `win` 在同一模块中的相对偏移固定。第一轮输入 15 个 `0` 和一个极大值。排序把初始的 `handler=&unique` 移入会被打印的 `nums` 区域，从输出中得到运行时 `unique` 地址：

```python
unique_runtime = leaked_value
win_runtime = unique_runtime - elf.symbols["unique"] + elf.symbols["win"]
```

第一轮可以出现重复元素，因为随后选择 `Y` 会重新把 `handler` 设为 `unique`。第二轮输入 15 个互不相同的小整数与 `win_runtime`。排序后最大值被移动到越界位置 `nums[16]`，即把 `handler` 改成 `win`。第二轮必须保持 16 个用户元素互不重复，否则后续重复检查会把指针覆盖为 `repeated`。

选择 `N` 退出循环后，`main` 执行 `handler()`，进入 `win()` 并获得 shell：

```text
greyhats{shUfFl3_tHe_Funt1on_pTr_oUt_5581d}
```

## 方法总结

这是“越界排序数据”和“相邻控制数据”组合产生的函数指针劫持。利用分成两轮：第一轮把已有函数指针搬进可见数组完成 PIE 泄漏，第二轮把计算出的 `win` 地址搬回函数指针。排序方向、极值位置和重复检查的执行顺序都必须逐项确认。
