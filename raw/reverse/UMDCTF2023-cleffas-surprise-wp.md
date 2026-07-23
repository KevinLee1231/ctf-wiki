# Cleffa's Surprise

## 题目简述

题目提供一个 macOS ARM64 Mach-O。主程序申请内存、复制一段 AArch64 shellcode、把页面改为可执行后跳转过去；shellcode 不输出任何内容，而是用一系列 `movk` 指令把 flag 分块写到栈上。程序还调用 `ptrace(PT_DENY_ATTACH)` 阻止调试器附加。

## 解题过程

在主函数中可以识别出以下执行链：

```text
ptrace(PT_DENY_ATTACH)
mmap(PROT_READ | PROT_WRITE)
memcpy(shellcode)
mprotect(PROT_READ | PROT_EXEC)
indirect call
```

调试时可以补丁掉 `ptrace` 调用，或在调用之后启动/附加调试流程。若不运行样本，也可以直接把复制源地址处的字节作为 AArch64 指令反汇编。

shellcode 重复使用四条 `movk` 拼出一个 $64$ 位值：

```asm
movk x1, #imm0
movk x1, #imm1, lsl #16
movk x1, #imm2, lsl #32
movk x1, #imm3, lsl #48
str  x1, [sp, #-offset]
```

每条 `movk` 替换寄存器中的一个 $16$ 位片段，`str` 再以小端序把八个字节写到栈上。四组数据分别写入
`sp-8`、`sp-16`、`sp-24` 和 `sp-32`。由于栈向低地址增长，按字符串顺序恢复时要从最低地址开始读取，并处理每个 $64$ 位字的小端字节序。

在最后一次 `str` 后检查 `sp-32` 到 `sp-1` 的内存，重新排列得到：

```text
UMDCTF{cleff4_l0v3s_5h3llc0d3}
```

后续 `svc` 并没有把这段内容正常打印出来，因此动态执行后“没有输出”是预期行为，栈内容才是目标。

## 方法总结

- 核心技巧：识别“复制到可执行页再间接调用”的 shellcode 装载器，并直接跟踪 shellcode 的内存副作用。
- AArch64 的 `movk` 适合分段构造立即数；连续的移位值 $0,16,32,48$ 往往代表拼接字符串或常量。
- 恢复栈字符串时必须同时考虑栈增长方向和小端序。
- 遇到 `PT_DENY_ATTACH` 不必绕复杂反调试，可静态反汇编 shellcode或补丁掉单个调用。
