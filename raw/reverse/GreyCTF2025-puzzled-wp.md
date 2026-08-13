# GreyCTF 2025 Puzzled Writeup

## 题目简述

附件是一个带大量干扰代码和反调试检查的 Linux ELF。程序读入一串由 `a` 至 `f` 组成的字符，将它们解释为魔方转动；只有最终六个面的九个色块都恢复为各自面编号时，才会读取 flag。

题目的主要障碍是从混淆后的程序中确认真实状态机、字母与转面的对应关系，以及硬编码初始魔方状态，而不是绕过某个内存漏洞。

## 解题过程

`cube[6][9]` 保存六个面。判定函数对每个 `face` 和 `pos` 检查：

```c
if (cube[face][pos] != face) {
    return 0;
}
```

因此目标状态完全明确：第 $i$ 个面的九个元素都必须等于 $i$。六个移动函数先旋转当前面的 $3\times3$ 数组，再交换四个相邻面的边条。`process_move` 给出的字符映射为：

```text
a -> Front
b -> Back
c -> Left
d -> Right
e -> Up
f -> Down
```

程序还包含两层反调试。第一层 fork 子进程并调用 `ptrace`，第二层读取 `/proc/self/stat` 找到父进程，再把经 `0x41` 异或隐藏的 `gdb`、`strace`、`ltrace`、`radare2`、`r2`、`x64dbg`、`lldb` 名称还原后进行匹配。这些检查只影响动态调试，不改变魔方转移规则；函数尾部的大段 NOP 和混淆器插入的垃圾分支也可从核心数据流中排除。

将硬编码初始状态与六个转面函数复制到一个小型状态模拟器，按常规魔方搜索或直接使用仓库给出的官方求解序列，即可得到：

```text
aabdddcccfddbbbddccbbcaaadcffbeeedddbcfdbbbeefff
```

把该序列送入随题发布的二进制，程序实际输出 `Success! Access granted.`，证明它确实把魔方恢复到 `is_solved()` 所要求的状态。远端随后读取真实配置中的 flag：

```text
grey{rub1ks_cub3_puzzle_with_lots_of_obfffffffffffuscation}
```

## 方法总结

面对体积很大的混淆二进制，应先围绕输入、状态更新和成功条件切出最小语义闭环。这里真正需要保留的只有初始 `cube`、六个 move 函数、字符分派和 `is_solved`；反调试与 NOP 膨胀只需识别其影响边界。把这部分逻辑抽成确定性的魔方状态机后，题目就从“分析整份混淆 ELF”降为“求一个有限状态变换序列”。
