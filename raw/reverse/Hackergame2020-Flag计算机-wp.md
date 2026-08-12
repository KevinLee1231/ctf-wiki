# Hackergame2020 Flag 计算机 WP

## 题目简述

附件是同时含 Windows PE 外壳和可运行 DOS stub 的混合程序。Windows 路径只弹出提示；真正的 flag 逻辑位于 DOS 实模式代码。程序把当前时间压缩到 $0\ldots58378$，作为线性同余序列参数，再经固定矩阵和异或表生成 30 字节输出。

## 解题过程

PE 文件的 DOS stub 通常只打印固定提示，但十六进制查看可发现这里充满代码。载入 IDA 时选择 MS-DOS executable，并把段关系校正为 `CS=DS=SS`；程序由 GCC 的 `.code16gcc` 生成，大量 `0x66` 是 operand-size override，使 16 位实模式代码可以使用 32 位操作数。若段基址差 `0x100` 未修正，IDA 会把数据引用解释到错误位置。

删除显示与延时逻辑后，核心等价于：

```c
a = get_time() % 58379;
x0 = 1103515245;

for (int j = 0; j < 15; ++j) {
    x0 = x0 * a + 12345678;  // uint32_t 自动模 2^32
    rand_gen[j] = x0;
}

for (int i = 0; i < 15; ++i) {
    for (int j = 0; j < 15; ++j) {
        matrix_out[i] += (matrix[i][j] * rand_gen[j]) & 0xffff;
        matrix_out[i] &= 0xffff;
    }
}

for (int i = 0; i < 30; ++i) {
    out[i] = aim[i] ^ matrix_out[i % 15];
}
```

`matrix` 的 225 个 16 位常量和 `aim` 的 30 个常量都可从修正版附件或官方源码提取。穷举空间只有 58379：

```python
for a in range(58379):
    candidate = recover(a, matrix, aim)
    if candidate.startswith(b"flag{") and candidate.endswith(b"}"):
        if all(0x20 <= byte < 0x7F for byte in candidate):
            print(a, candidate)
```

生成题目时使用的参数是 `a=26141`，修正版附件最终恢复：

```text
flag{g3tfl4g_0p3r4t1ng_syst3m}
```

官方 17 张图片主要是 PE 头、IDA 设置和反汇编文本；关键段设置、前缀含义、公式与常量来源已完整转写，因此不保留序号截图。

## 方法总结

混合格式文件不能只按最外层 PE 处理；DOS stub 也可能是完整程序。实模式逆向时必须先校准段基址和操作数宽度，再恢复算法。时间只提供 58379 个候选，穷举比试图猜真实运行时刻更稳定，并可用 flag 格式与可打印性做强校验。
