# UIUCTF 2023 Fast Calculator Writeup

## 题目简述

题目提供一个静态链接的命令行计算器。输入两个操作数和运算符后，程序会计算结果；只有结果恰好等于 `8573.8567` 时，才会对内置密文执行 368 次浮点运算，并按每次结果是否通过 `gauntlet()` 判定翻转密文中的对应位。

关键不在寻找另一组“秘密算式”，而在编译选项。发布版使用 `-O0 -ffast-math`，而 `gauntlet()` 又依赖 IEEE 754 的负数、NaN 与无穷判断。`-ffast-math` 允许编译器假设这些特殊值不会出现，破坏了程序本来用于解密的判定逻辑，因此发布版只会解出假 flag。

## 解题过程

先观察 `main` 中进入解密分支的条件：

```c
if (result == 8573.8567) {
    for (int i = 0; i < len_operations; i++) {
        if (gauntlet(calculate(operations[i]))) {
            /* 翻转密文中的对应位 */
        }
    }
}
```

因此入口可以直接用不改变数值的加法满足：

```text
8573.8567 + 0
```

发布版随后报告翻转 119 位，并给出：

```text
uiuctf{This is a fake flag. You are too fast!}
```

继续还原判定位。`calc.h` 中三个包装函数分别检查符号位、NaN 和无穷：

```c
bool isNegative(double value)  { return signbit(value); }
bool isNotNumber(double value) { return isnan(value); }
bool isInfinity(double value)  { return isinf(value); }

bool gauntlet(double result) {
    return isNegative(result) || isNotNumber(result) || isInfinity(result);
}
```

这组判断需要保留 IEEE 754 特殊值语义。例如某些静态运算会产生 `-0.0`、NaN 或正负无穷；只要结果为负数或特殊值，程序就应翻转一个密文位。但 `-ffast-math` 包含有限数学、无符号零等激进假设，编译器可以把 `isnan()`、`isinf()` 等路径折叠掉，并使负零相关判断失真。于是同一份操作表在发布版和正常浮点语义下生成不同的位流。

题目 `Makefile` 已明确展示两种构建：

```makefile
fast:
	$(COMPILER) calc.c -o calc -ffast-math $(FLAGS)

solve:
	$(COMPILER) calc.c -o calc-SOLUTION $(FLAGS)
```

所以可以删除 `-ffast-math` 重新编译，或直接使用仓库中按 `solve` 目标生成的 `calc-SOLUTION`。再次输入同一算式，正确语义下会翻转 244 位：

```text
$ ./calc-SOLUTION
Enter your operation: 8573.8567 + 0
Result: 8573.856700

Correct! Attempting to decrypt the flag...
I calculated 368 operations, tested each result in the gauntlet, and flipped 244 bits in the encrypted flag!

uiuctf{n0t_So_f45t_w1th_0bscur3_b1ts_of_MaThs}
```

如果只有发布二进制而没有源码，也可在反编译器中恢复 368 个操作和密文字节，再用遵循 IEEE 754 的实现重放计算与位翻转；本质仍是修复 `gauntlet()` 的语义，而不是暴力搜索输入。

## 方法总结

这道题利用了 `-ffast-math` 与 IEEE 754 特殊值之间的语义差异。逆向时既要看控制流，也要检查构建信息和编译器优化假设；当源码中的 NaN、Infinity、负零检查与反编译结果明显不符时，应优先怀疑编译选项。恢复正常浮点语义后，原输入、操作数组和密文都不需要改变，正确 flag 会自然解出。
