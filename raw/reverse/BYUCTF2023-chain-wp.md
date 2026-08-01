# BYUCTF 2023 - Chain

## 题目简述

附件是 ARM ELF。程序要求输入长度为 45，其中最后一字节是换行；前 44 个字符分别与从固定基址和偏移表取出的字节异或，再与 `correct` 数组比较。

## 解题过程

源码/反编译逻辑等价于：

```c
for (int i = 0; i < 44; i++) {
    char c = *(base + offsets[i]);
    if ((buf[i] ^ c) != (char)correct[i]) fail();
}
```

异或可逆，所以每个字符独立恢复：

```python
flag = bytes(correct[i] ^ memory[base + offsets[i]] for i in range(44))
```

`base = 0x105ac`，`offsets` 与 `correct` 都是静态数组；从二进制对应虚拟地址读取字节即可。恢复为：

```text
byuctf{1_h0p3_ARM_wasn't_t00_b4d_0f_4_tw1st}
```

## 方法总结

不要把乱序偏移误认为“链式依赖”：每个输出只依赖一个静态字节和一个输入字符，44 个位置可并行求解。先把比较方程写成代数形式，通常比单步跟完整循环更清晰。
