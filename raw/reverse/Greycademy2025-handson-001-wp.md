# handson-001

## 题目简述

题目位于赛事 Pwn 训练目录，但没有要求构造内存破坏或控制流劫持；目标是用 GDB/Pwndbg 回答固定内存、指令和栈地址问题。三题全部答对后程序才启动 shell，因此决定性障碍是动态分析，归入 Reverse。

## 解题过程

第一关要求读取 `0x404500` 的 64 位值：

```gdb
x/gx 0x404500
```

结果为 `0xd00df00d`。第二关要求反汇编 `0x401399`：

```gdb
x/i 0x401399
```

应提交不带分号的 `pop rdi`。

第三关打印的“栈末尾”地址是 `rbp+0x10`，而反汇编显示 `stack.attempt` 位于 `rbp-0x30`，两者相差 `0x40`。远端启用 ASLR，不能照抄本地绝对地址；应实时计算：

```python
printed_end = int(input("end address: "), 16)
print(hex(printed_end - 0x40))
```

提交该地址后，程序比较输入值与 `&stack.attempt`，三关通过便执行 `/bin/sh`。读取 `flag.txt` 得到：

```text
flag{gdb_m4st3r}
```

## 方法总结

动态栈题应提取稳定偏移，而不是依赖一次运行的绝对地址。`x/gx` 用于查看 8 字节内存，`x/i` 用于按指令解释地址；结合函数栈帧中的 `rbp` 相对位置，就能把本地 GDB 观察迁移到启用 ASLR 的远端。
