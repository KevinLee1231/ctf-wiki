# Watchdog KOTH

## 题目简述

这是 `Watchdog` 的现场 King of the Hill 版本。提交物只能是一份 `.wdog`，软核必须通过 UART 输出完整 flag 和换行；成绩按完成所需时钟周期排名。题面给出的作者参考成绩约为 `0x163` 周期，但仓库没有公开该优化 payload，也没有保存最终赛事冠军结果。

## 解题过程

先以基础题 payload 建立正确性基线：关闭 watchdog，从 `0x20000000` 读取 flag，向 `0x10000000` 的 UART 数据寄存器发送，并以 `\n` 结束。`.wdog` 仍使用：

```text
"WDOG" || uint32_le(payload_length) || raw_rv32i_payload
```

基础题为可靠性在 `0x10000200..0x100002fc` 做了 64 次 MMIO 写入，并使用 C 循环、函数调用和逐字符 UART ready 轮询。这些都是明显的周期优化点，但在确认替代方案前不能直接删除。

KOTH 的优化顺序应为：

1. 用 bitstream 仿真、差分实验或最小 MMIO 探测定位真正的 watchdog 控制地址，把 64 次窗口写零缩减为必要的单次或少数写操作。
2. 用手写 RV32I 从 `_start` 直接执行，消除 C 运行时、函数序言/尾声和通用循环开销。
3. 依据固定 flag 长度展开读取和发送，减少分支与索引更新；每次仍保持正确的小端字节顺序。
4. 测量 UART ready 的时序。只有在硬件保证相邻写不会丢字符时才能减少轮询，否则应通过调度其他指令填充等待，而不是盲删状态检查。
5. 确保最后一个字符后还有 ASCII 换行 `0x0a`；少一个字节即使周期更低也不构成有效成绩。

一个用于比较的伪汇编骨架为：

```asm
_start:
    lui  t0, WATCHDOG_REG_HI
    sw   zero, WATCHDOG_REG_LO(t0)

    lui  t1, 0x20000          # FLAG_BASE
    lui  t2, 0x10000          # UART_BASE
    # unrolled loads / byte extraction / UART writes
    # ...
    li   t3, 10
    sw   t3, 0(t2)
hang:
    j    hang
```

每个候选 payload 都应先在本地题目 bitstream 上核对 UART 的逐字节输出：

```text
grey{mmio_fuzzz}\n
```

再记录从 `RUNNING` 到完整换行之间的周期数。基础题的通用 C 解法是一份有效但较慢的 KOTH 基线；`0x163` 仅是题面公布的作者参考，不应当写成已复现成绩。

## 方法总结

KOTH 优化必须建立在可重复的周期计数和字节级正确性之上。最大的结构性浪费是广泛 MMIO fuzz，其次才是 C 循环和 UART 发送开销。仓库足以说明提交格式、内存映射、有效输出和优化方向，但没有公开最优 `.wdog`，因此本文保留可验证基线与优化方法，不虚构最终最短周期。
