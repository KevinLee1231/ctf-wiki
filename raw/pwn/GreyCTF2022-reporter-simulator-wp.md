# GreyCTF2022 - Reporter Simulator

## 题目简述

程序以 `nlen * sizeof(Report)` 计算分配大小，并只检查乘积是否不超过 `0x1000`。`nlen` 可令 64 位乘法回绕成很小的值，但后续清零、拷贝与索引仍使用未经约束的逻辑数量，形成堆越界读写。

## 解题过程

选择一个使乘积模 $2^{64}$ 落入允许范围的巨大 `nlen`。这样实际分配很小，而索引换算和对象初始化可跨越相邻 chunk。先通过越界显示/字符串字段泄露堆地址，再读取 unsorted-bin 指针确定 libc 基址。

```python
MASK = (1 << 64) - 1
nlen = choose_wrap_value(report_size, wanted_alloc)
assert (nlen * report_size) & MASK <= 0x1000
create_report_array(nlen)

heap = leak_adjacent_chunk()
libc.address = leak_unsorted_bin() - main_arena_offset
```

利用越界写伪造相邻 Report 的字符串指针，使一次编辑落到 `__free_hook`，写入 `system`；再释放保存 `/bin/sh` 的对象：

```text
grey{pl5_f0llow_resp0nsib1e_d1sclo5ur3_12a33m}
```

## 方法总结

“乘法后检查”无法防止整数溢出，安全写法应先检查 `count > SIZE_MAX / element_size`。利用分析要区分回绕后的真实分配字节数与后续仍使用的逻辑元素数，并逐处核对每个乘法是否采用相同位宽。
