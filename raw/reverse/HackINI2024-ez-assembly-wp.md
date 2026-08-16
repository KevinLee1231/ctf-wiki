# HackINI2024 ez assembly

## 题目简述

题目给出一段 x86-64 NASM 源码。程序逐字节读取候选 flag，对每个字符先取负数，再左移 2 位，并与 `output` 数组中的 64 位常量比较。目标是逆转这两个运算，恢复原始字符。

## 解题过程

预期检查逻辑为：

```asm
movzx rax, byte [input + rcx]
neg rax
shl rax, 2
mov rbx, [output + rcx * 8]
cmp rax, rbx
```

对字符值 $c$，数组中保存的是 64 位二进制补码：

$$
y=-4c\pmod {2^{64}}
$$

先把无符号常量解释为有符号 64 位数，再取负并除以 4，即可得到字符：

```python
data = [
    0xfffffffffffffe34, 0xfffffffffffffe60, 0xfffffffffffffe6c,
    0xfffffffffffffe50, 0xfffffffffffffe50, 0xfffffffffffffe4c,
    0xfffffffffffffe7c, 0xfffffffffffffe30, 0xfffffffffffffe6c,
    0xfffffffffffffe34, 0xfffffffffffffe14, 0xfffffffffffffe24,
    0xfffffffffffffe60, 0xffffffffffffff40, 0xfffffffffffffe84,
    0xfffffffffffffe34, 0xffffffffffffff30, 0xffffffffffffff3c,
    0xfffffffffffffef0, 0xfffffffffffffe84, 0xffffffffffffff30,
    0xffffffffffffff2c, 0xffffffffffffff2c, 0xfffffffffffffe4c,
    0xfffffffffffffe78, 0xfffffffffffffed0, 0xfffffffffffffe1c,
    0xfffffffffffffe84, 0xffffffffffffff3c, 0xffffffffffffff2c,
    0xfffffffffffffe84, 0xfffffffffffffe60, 0xffffffffffffff30,
    0xfffffffffffffeb8, 0xfffffffffffffef0, 0xffffffffffffff04,
    0xfffffffffffffe0c,
]

flag = []
for value in data:
    signed = value if value < (1 << 63) else value - (1 << 64)
    flag.append(chr((-signed) // 4))

print("".join(flag))
```

对全部 37 个常量执行后得到：

```text
shellmates{wh0_s41D_455mbLy_15_h4RD?}
```

发布源码存在一处索引错误：实际写成了 `[output + rcx]`，每次只前进 1 字节，而 `output` 的元素是 8 字节 qword。按原样汇编后，第二次比较起就会读取跨元素的错位数据，无法接受官方 flag。把索引改为 `[output + rcx * 8]` 后才与官方解法和常量表一致。因此上面的 flag 是题目明确的预期结果，而不是未修正源码可以直接验证的结果。

## 方法总结

逆向汇编算术时要同时考虑位宽、补码和数组步长。`neg` 与乘 4 都可直接逆转，但 64 位常量应先做有符号解释。源码级题目也可能包含实现笔误；当循环索引与数据声明的元素宽度冲突时，应把“预期算法”和“发布代码实际行为”分别记录，不能静默修正后声称原程序可用。
