# GlacierCTF 2025 awesomeNESs

## 题目简述

附件是一张 Mapper 0（NROM）的 NES ROM。界面允许输入 39 个 flag 内部字符，6502 校验器为每个位置从输入字符推导一个函数索引，再让同一函数分别处理两条字节流，并与两组 expected bytes 比较。

题目刻意利用 NMOS 6502 的 `JMP ($xxFF)` 跨页读取缺陷、越界函数索引和非法指令 `ARR`。官方 `WRITEUP.md` 为空，但汇编源码、ca65 debug symbols、expected-table 生成器和单解验证器足以完整还原预期解法。

## 解题过程

### 1. 还原输入字符到 tile index 的映射

游戏保存的不是 ASCII，而是图块编号。工具脚本给出了完整 `alpha_table`，例如：

```text
a -> 0, A -> 1, b -> 2, B -> 3, ...
1 -> 52, ... 0 -> 61, _ -> 62
```

`NUM_FLAG_CHARS` 为 `$27`，即只校验 `gctf{...}` 中间的 39 个字符。对第 $i$ 个输入值 $c_i$，程序按照 `flag_checker_input_offset_op[i]` 选择 ADC、SBC 或 EOR，再与 `flag_checker_input_offset[i]` 组合，得到函数索引 $y_i$。

### 2. 分析间接跳转与两种异常路径

函数表包含 ADC、AND、SBC、EOR、ORA、输入相关运算，以及 ASL、LSR、ROL、ROR。delegate 将索引乘 2，从表中取 16 位地址写入 `flag_checker_jmp_buf`，最后执行：

```asm
jmp (flag_checker_jmp_buf)
```

这个 buffer 被故意放在页尾 `$03FF`。真实 6502 的间接跳转硬件 bug 会从 `$03FF` 取低字节，却从 `$0300` 而非 `$0400` 取高字节。初始化代码把 `$0300` 设置为 `>check_fn_adc`，即 `$87`；正常函数都布置在 `$87xx` 页内，因此仍能调用，但两个特殊索引会产生异常控制流：

- 索引 0 的表项原本指向 `$86F0`，但高字节 `$86` 被页面回绕忽略，实际跳到 `$87F0` 的流数据 `$38,$6B,$55,$60`，即 `SEC; ARR #$55; RTS`。`$6B` 是非官方 6502 指令，语义近似先 `AND #imm` 再带 carry 右旋。
- 某位置推导出越界索引 `$35`，从函数表后的 stream1 读到低高字节 `$F3,$86`。若没有硬件 bug，它会跳到 `$86F3` 的 `check_brk`；实际高字节仍被替换成 `$87`，所以落到 `$87F3` 的单独 `RTS`，直接返回。

因此用普通 6502 反汇编器把所有函数索引限制在表长内，或用不实现 NMOS page-wrap/非法 opcode 的模拟器，都会得到错误结果。

### 3. 用 CPU 模拟逐位置恢复输入

官方工具展示了可靠的仿真环境：

1. 解析 16 字节 iNES header；
2. 将 PRG ROM 按 NROM 映射到 `$8000`；
3. 从 ca65 `.dbg` 文件读取 `flag_input`、`flag_checker`、两个失败点和成功点等符号；
4. 把 wraparound 高字节写入对应 RAM；
5. 使用支持 NMOS 行为的 `py65emu` 从 `flag_checker` 开始单步执行。

可按前缀逐字节搜索：已恢复的前缀写入 `flag_input`，当前位置遍历 `alpha_table` 中允许的字符，后缀填任意值；每次把 PC、A/X/Y 和 flags 重置，从 checker 运行。如果在当前位置进入 stream1/stream2 fail，则丢弃；能推进到下一字符的候选保留。39 个位置全部通过后，还要验证两个 rolling `state_checksums` 与 `expected_state_checksums` 相等。

仓库的 `check_single_solution.py` 会进一步将每一位替换成所有其他字符，确认没有第二组输入也能到达成功状态。汇编注释中的明文只应作为最后交叉校验，恢复结果为：

```text
gctf{0op5_wr0ng_jmp_t0_i1l3g4l_0pc0d35_s0rry}
```

## 方法总结

本题的重点不是把 expected 数组简单异或，而是精确复现 6502 的执行语义：状态寄存器和 carry 会影响 ADC/SBC/rotate，`JMP ($xxFF)` 有页面回绕，`ARR` 属于非法 opcode，越界索引又能跳入看似普通的数据。最稳妥的流程是先从 ROM 和 debug symbols 建立最小 CPU/MMU，再做逐位置差分搜索，最后用 rolling checksum 和单解枚举双重验证。
