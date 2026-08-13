# GreyCTF2024 RF Signal Analysis WP

## 题目简述

附件 `rf_dump.dat` 是 7,761,600 个小端 `float32` 实数采样。信号使用开关键控（OOK）和 Manchester 编码：载波“有/无”表示高低电平，每个数据位由两个能量相反的半符号组成。

## 解题过程

先按小端单精度读取数据。观察短时均方能量可见，开头约 1,129,000 点是静音；有效区的能量状态每约 8,820 点切换一次。取 `start = 1128900`、`half = 8820`，按半符号计算均方能量并以 0.1 为阈值：

```python
import numpy as np

x = np.fromfile("rf_dump.dat", dtype="<f4")
start, half = 1128900, 8820

energy = np.array([
    np.mean(x[i:i + half] ** 2)
    for i in range(start, len(x) - half + 1, half)
])
cells = (energy > 0.1).astype(np.uint8)

bits = []
for i in range(0, len(cells) - 1, 2):
    pair = tuple(cells[i:i + 2])
    if pair == (1, 0):
        bits.append(1)
    elif pair == (0, 1):
        bits.append(0)
    else:
        break

raw = bytes(
    sum(bits[i + j] << (7 - j) for j in range(8))
    for i in range(0, len(bits) - 7, 8)
)
print(raw)
```

312 个 Manchester 数据位还原为 39 字节：

```text
aa 67 72 65 79 7b 33 34 73 79 5f 30 6e 5f 30 66 ...
```

首字节 `0xaa` 是同步前导，末尾是填充空格；中间 ASCII 解码为：

```text
grey{34sy_0n_0ff_k3y1n9_M0dul4710n!}
```

## 方法总结

OOK 的第一步不是直接对载波逐周期解调，而是把短时能量降成二值包络。测出稳定的半符号长度后，Manchester 的互补二元组还能同时提供时钟和数据；最后要去除同步前导与尾部填充，不能把它们误并入 flag。
