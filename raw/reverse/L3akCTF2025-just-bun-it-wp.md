# L3akCTF 2025 Just Bun It Writeup

## 题目简述

题目给出无源码二进制 `runme` 和 `input.txt`。每行包含一个很大的非负整数 $n$ 与一个 $[0,1)$ 内的小数 $x$；执行 `runme n x` 会输出一个字符，依次处理全部行即可得到 flag。

直接让程序循环 $n$ 次不可行，因为后面的 $n$ 有数百位。决定性障碍是逆向小数变换并利用其有限状态周期，因此本文按 Reverse 归档。

## 解题过程

### 还原单步变换

静态分析可以定位到字符表：

```text
0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ{_}
```

引用该字符串的主流程先迭代一个小数变换，再计算：

```text
index = int(100 * value) % 65
output = charset[index]
```

IDA 反编译得到的单步函数为：

```python
def step(x):
    if 0 <= x < 0.25:
        y = x + 0.55
    elif 0.25 <= x < 0.55:
        y = x - 0.25
    elif 0.55 <= x < 0.75:
        y = x + 0.20
    elif 0.75 <= x < 1.0:
        y = x - 0.50
    else:
        y = x
    return custom_round(y * 1000.0) / 1000.0
```

其中 `custom_round` 是半入舍入，而不是 Python 对恰好 `.5` 采用的银行家舍入：

```python
import math

def custom_round(v):
    base = math.floor(v)
    return base + (v - base >= 0.5)
```

这与二进制中“先乘 1000、调用取整函数、再除 1000”的实现一致。

### 利用尾链和循环

每一步都将结果量化到三位小数，所以可能状态最多约 1000 个。对任意初始 $x$ 连续应用 `step`，必然先经过一段不重复的尾链，然后进入循环。记录状态第一次出现的位置即可得到二者：

```python
def find_cycle(x):
    states = []
    seen = {}
    while x not in seen:
        seen[x] = len(states)
        states.append(x)
        x = step(x)

    start = seen[x]
    return states[:start], states[start:]

def nth_value(n, x):
    tail, cycle = find_cycle(x)
    if n < len(tail):
        return tail[n]
    return cycle[(n - len(tail)) % len(cycle)]
```

这样只需要对巨大十进制整数做一次模运算，不需要真正迭代 $n$ 次：

```python
charset = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ{_}"

flag = []
for line in open("input.txt", encoding="utf-8"):
    n_text, x_text = line.split()
    value = nth_value(int(n_text), float(x_text))
    flag.append(charset[int(value * 100) % len(charset)])

print("".join(flag))
```

运行后得到：

```text
l3ak{bun_thought_binary_lifting_was_needed_turns_out_f_was_cyclic_after_all}
```

题面已特别说明本题前缀为小写 `l3ak`，因此这里不是抄写错误。

## 方法总结

本题用极大的迭代次数制造“需要高阶加速”的假象，但三位小数量化已经把连续空间压缩成有限状态机。有限状态上的确定性函数一定产生尾链加循环，找到首次重复点后即可把任意大 $n$ 化为一个很小的循环下标。

逆向数值程序时必须同时保留分段边界和舍入规则。`0.25`、`0.55`、`0.75` 的归属以及 `.5` 的处理方式都会改变后续轨道；若直接使用语言默认的 `round`，即使周期方法正确，也可能得到错误字符。
