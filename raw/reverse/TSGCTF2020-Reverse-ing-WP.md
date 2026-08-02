# TSGCTF2020 Reverse-ing WP

## 题目简述

程序读取 37 字节输入并判断是否正确，但静态反汇编看到的指令顺序并不可信。二进制中的 `reverse` 函数每次都会原地翻转 `_start` 到 `end` 之间的代码区；主流程又在几乎每条有效指令后调用一次 `reverse`。因此同一片代码在正向和反向布局之间交替，只有跟随运行时状态才能看到实际执行序列。

## 解题过程

先在 GDB 中于 `reverse` 返回后的稳定位置下断点，并准备 37 字节占位输入。每次程序执行 `call rbx` 时，`rbx` 都指向 `reverse`；让该调用完整执行后再继续单步，就能越过自修改过程，观察下一条真正的逻辑指令。

官方脚本在 `reverse+52` 处断下，循环检查 `$rip`：遇到访问 `[rdx+0x600194]` 的两类指令时，记录当前内存字节；遇到 `call rbx` 就继续运行到断点。关键提取逻辑为：

```python
if "xor    cl,BYTE PTR [rdx+0x600194]" in inst:
    xs.append(get_byte_at("$rdx+0x600194"))
elif "add    cl,BYTE PTR [rdx+0x600194]" in inst:
    ys.append(get_byte_at("$rdx+0x600194"))
```

将交替翻转后的指令重新按执行顺序排列，可以手工化简为：

```c
read(0, flag, 37);
unsigned char bad = 0;
for (int i = 0; i < 37; i++) {
    bad |= (unsigned char)((flag[i] ^ A[i]) + B[i]);
}
puts(bad ? "wrong" : "correct");
```

由于所有运算都在 8 位寄存器 `cl` 上进行，正确条件是：

$$
((f_i\oplus A_i)+B_i)\bmod 256=0.
$$

所以逐字节逆运算为：

$$
f_i=(-B_i\bmod 256)\oplus A_i.
$$

程序从末尾向前处理输入，脚本需要把逐次观察到的结果逆序拼接：

```python
answer = ""
for a, b in zip(xs, ys):
    answer = chr(((256 - b) & 0xff) ^ a) + answer
```

恢复出的 flag 为：

```text
TSGCTF{S0r3d3m0_b1n4ry_w4_M4wa77e1ru}
```

## 方法总结

这类自修改程序不能只依赖一次静态反汇编，因为字节在每次调用后都会改变语义。可靠做法是先识别状态转换点，本题即 `call rbx -> reverse`，再在每个稳定状态记录真正执行的操作数和数据。最终校验只是逐字节 XOR、加法与 OR 汇总；难点全部来自代码不断反转造成的控制流表示。调试时应明确寄存器宽度，`cl` 的溢出意味着公式必须按模 256 计算。
