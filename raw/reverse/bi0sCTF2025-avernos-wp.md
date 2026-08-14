# bi0sCTF 2025 - avernos WP

## 题目简述

题目位于 `bi0sCTF-main/2025/REV/avernos`，核心是一个混淆明显的验证题。`README.md` 明确描述了题目链路：程序先通过 mixed-mode（混合模式）进行输入与反调试处理，再进入自定义 VM，对输入 flag 做 4 段分步校验。官方仓库给出的目标输出为 `bi0s{m1x_m0d3_4_fun_w4y_to_h1de_s7uff}`，并给出 4 段校验路线（CRC32 pair、RC4、滚动异或、常量尾部）。

`admin/avernos.cpp` 与 `admin/solve.py` 形成了最直接的“题目实现 + 复现脚本”对应关系：
- 源码层面定义了 mixed-mode C++/CLI 的入口流转、SEH trampoline、字节码 VM 与校验逻辑。
- 脚本层面给出了可复原 flag 的分段生成顺序。
这使得 WP 可以不依赖“猜测式复现”，只基于源码链路给出可执行验证结论。

## 解题过程

第一步先按启动链路确认 anti-debug 与流程劫持。`main` 里通过 `fopen` 打开一条不存在的随机路径（`/home/.../flag.txt`）获得 `bogus`，并围绕 `func2` 建立 `__try/__except`；异常时会进入 `sussy()` 与 `Trampoline1`。这条链路的目的是把控制流引入异常处理分支，而不是直接走线性输入路径。

`sussy()` 使用 `CreateFileA` 对当前模块路径做检测，结合后续 `Trampoline1/Trampoline2` 的除零行为构成 anti-debug/SEH trampoline：两个函数分别触发异常并互相转移到下一层 handler，形成故意“混淆控制流”。该设计符合 `README` 与题目注释对 anti-debug 的描述。

在 SEH 结构解开之后，程序会跨 managed/unmanaged 边界切换到托管世界。`avernos.cpp` 中通过 `#pragma managed`/`#pragma unmanaged` 组合，以及 C++/CLI 的 `System::String^`/`Console::ReadLine()`/`System::Text::RegularExpressions::Regex`，`func6()` 会对用户输入做 `bi0s{...}` 形态校验并写入 `global_flag`，再回到 native 侧继续执行，`func8` 将托管输入注入到原生输入缓冲。这个 managed-unmanaged 过渡是完整链路的关键，不经过它无法进入真实校验。

第二步确认 VM。`admin/avernos.cpp` 定义了固定长度的 `instruction`，包含计数器、密钥、加密 opcode、3 个参数；`func5()` 把字节码装载进 `vm_instructions`，`run_vm()` 初始化 VM 状态后按指令计数器取指、执行、跳转。`func1()` 里通过 `decrypt_embedded_instructions()` 先对指令流做 nibble 变换与异或解码，再解码 opcode 和参数，执行 `HALT` 前后分别与状态分支联动。实际校验在 VM 中对 32 字节输入进行逐段比较，终点通过 `func18()` 返回的指针（代码内固定相关值）访问 `vm_memory` 决定是否通过。该结构与 `README` 的“VM checks flag in 4 parts”一致。

第三步用官方 `admin/solve.py` 做四段还原（不写新脚本、不改变题目逻辑）。

1. **Part 1（CRC32 pairs）**：脚本的 `hash()` 逐位右移并按 LSB 决定是否异或固定常数 `0xD90B8320`。按可打印范围 32~127 进行暴力拼出 8 字节前缀，目标常量为 `0xcc9d08cb/0xf6a29795/0x5c12d754/0x21563cf9`。
2. **Part 2（RC4）**：使用长度 16 的改造 RC4，密钥字节为 `DEADBEEFCAFEBABE`，对密文 `56 3d 5c 64 7e 6c 5f 7e` 解码。
3. **Part 3（rolling XOR）**：对 `77 66 40 6b 70 40 77 6d` 做 2 字节滑窗异或，密钥 `43 5c`，覆盖式地逆向还原 8 字节。
4. **Part 4（constant bytes）**：直接拼接 `64 65 5f 73 37 75 66 66`（等价文本 `de_s7uff`）。

将 4 段拼接后得到 `m1x_m0d3_4_fun_w4y_to_h1de_s7uff`，再按 `bi0s{...}` 封装即 flag 完整串。

我在本地执行了基于同一逻辑的核验（避免依赖 `pwn`）验证最终结果：

`bi0s{m1x_m0d3_4_fun_w4y_to_h1de_s7uff}`

该结果与官方仓库 `README.md`、`admin/flag.txt` 的给定 flag 一致，也与 `admin/avernos.cpp` 的 VM 校验路径一致。[出题人题解](https://blog.bi0s.in/2025/07/17/RE/avernos-bi0sCTF2025/)同样确认了 mixed-mode、SEH、VM 和四段校验框架；决定性机制已经写入本文，外链只用于核对原始说明。

## 方法总结

题目核心不是“花哨算法”，而是通过 C++/CLI 混合态把程序行为切分到 managed 与 unmanaged 两边，靠 SEH trampoline 混淆执行入口，再把真正校验放入字节码 VM。反向时应先把流程层拆清：

1. 先恢复受控控制流（anti-debug + trampoline）。
2. 确认输入从 managed 回到 native 的实际数据路径。
3. 定位 VM 与解码器（`instruction`、opcode 解密、vm loop）。
4. 按 `solve.py` 的四段校验逐段计算并拼接。

这种顺序保证每一步都能“可复核”：源码给出机制，脚本给出输入分层，最终 flag 与仓库声明值严格对齐。
