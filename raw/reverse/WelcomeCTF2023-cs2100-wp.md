# CS2100

## 题目简述

服务端实现了一个自定义栈式虚拟机，并要求提交一段程序。虚拟机初始栈中放入两个随机 32 位整数；同一程序连续通过 100 组测试、使栈顶等于两数乘积后才返回 Flag。

指令集只有 `PUSH`、`POP`、`DUP`、`SWAP`、`SHIFT`、`NAND`、`JE` 和 `JNE`。题目的主要障碍是理解并使用受限 VM 指令实现整数乘法，因此归入 Reverse。

## 解题过程

先确认关键语义：

```text
NAND      弹出 a、b，压入 ~(a & b)
SHIFT     弹出 a、b，压入 b << a
JE n      弹出条件；为 0 时跳到第 n 条指令
JNE n     弹出条件；非 0 时跳到第 n 条指令
SWAP x y  交换从栈顶向下第 x、y 个元素
```

虽然只有 `NAND`，但它是函数完备的：

$$
\operatorname{NOT}(x)=\operatorname{NAND}(x,x),
$$

$$
x\land y=\operatorname{NOT}(\operatorname{NAND}(x,y)).
$$

进一步组合即可得到 OR、XOR 和测试某一位所需的掩码逻辑。官方 135 行程序采用移位加法乘法：维护 `result`、`multiplicand` 和 `multiplier`，检查乘数当前位，必要时把被乘数加入结果，然后令被乘数左移一位并继续下一位。

对应的高层伪代码是：

```python
result = 0
while multiplier != 0:
    if multiplier & 1:
        result += multiplicand
    multiplicand <<= 1
    multiplier >>= 1
return result
```

VM 没有直接加法和右移，因此官方程序用 NAND 构造逐位全加器，并通过循环扫描位位置完成等价操作；`SHIFT` 用于产生 $1\ll i$ 和被乘数的左移。提交时第一行是程序行数 `135`，后接全部指令。服务端对 100 组随机输入均验证通过后输出：

```text
greyhats{1_4p0l0g1z3_f0r_th3_n3xt_ch41}
```

## 方法总结

- 核心技巧：在仅有 NAND、移位、栈操作和条件跳转的 VM 中搭建布尔门、全加器与移位加法乘法。
- 识别信号：服务端把用户输入编译成自定义字节码，并以随机测试检查某个算术函数。
- 复用要点：先把指令语义抽象成宏，再实现高层算法；直接手写长字节码很容易出现栈深度和跳转编号错误。
