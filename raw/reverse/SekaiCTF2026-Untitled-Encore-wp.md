# Untitled Encore

## 题目简述

附件 `untitled-encore.exe` 是一个带谱面编辑界面的 Windows 程序，内部嵌入了一份 ELF64-BPF。C++ 外壳会校验 20 个音符组成的 40 字节 chart，解释 ELF 中的 eBPF 程序和一层自定义字节码，再把两边算出的 32 字节 key 作一致性检查；只有全部通过，才能解密最终 flag blob。

因此重点不是实际“游玩”谱面，而是把 PE 外壳、ELF/eBPF 和内层四字节 VM 分开还原。

## 解题过程

### 提取嵌入工件

官方 `solve.py` 在 EXE 中搜索 `7f 45 4c 46 02 01`，逐个候选解析为 64 位小端 ELF，直到其 `.rodata` 中出现 14 字节魔数：

```text
d6 27 91 a8 4c 03 f5 7e b9 10 6a c2 35 88
```

其后容器格式依次为版本号 `2`、一字节密文长度、两字节小端 bytecode 长度、密文和 bytecode。公开 EXE 中的内层 bytecode 长 388 字节。PE 本身还包含以
`9e 41 d4 72 0b c9 5a ef 36 a1 80 2d f7 64 b8 13`
标记的 C++ 最终密文 blob。

### 求解四字节 VM

每条内层指令为四字节 `(op, a, b, c)`，但磁盘上的第 $i$ 个槽会与当前位置 $pc$ 派生的单字节密钥异或：

$$
\begin{aligned}
k_0&=(17pc+0xa3)\bmod256,&
k_1&=(29pc+0x11)\bmod256,\\
k_2&=(31pc+0x7b)\bmod256,&
k_3&=(37pc+0xc5)\bmod256.
\end{aligned}
$$

这里 $pc$ 是 bytecode 中的字节偏移，每次增加 4。解码后只有五种操作码：

```text
0x21 OP_BYTE
0x44 OP_NOTE
0x62 OP_PROOF
0x8b OP_KEY
0xf0 OP_DONE
```

VM 状态从 `0x31c3f00d` 开始。`OP_BYTE` 依次确认 8 字节 selector
`25 00 39 04 1d 6a e5 7f`
和小端 chart 长度 40；`OP_NOTE` 约束第 $a$ 个音符的两个字节。

chart 共 20 组 `(x, delta)`。其中 `x` 打包了 lane、kind、flick 和 parity，合法域满足 `lane <= 4`、`kind <= 2`；`delta` 位于 $[3,16]$。对每个 `OP_NOTE`，只需在这个小域中枚举 `(x, delta)`，代入状态更新函数并要求两个检查字节等于指令中的 `b`、`c`。若后续冲突就回溯，无需枚举全部 $40$ 字节。

chart 完整后，`OP_PROOF` 从 chart 和滚动状态生成 32 字节 proof，`OP_KEY` 再由 chart、proof 和状态生成 32 字节期望 key，`OP_DONE` 结束。实际恢复结果为：

```text
chart = 000c01070a072305140d0b062a06010b1008640408090105320a03050c0f6203100c740649070309
proof = 33d263e1008c774e67812c7fb6567b2eb4986d3b028ce0b88146cdbd7f796e9b
```

### 通过 eBPF 与 C++ 双重校验

送入 eBPF 的 calldata 为：

```text
selector[8] || u32le(40) || chart[40] || proof[32]
```

C++ 外壳解释 ELF `.text` 中的 eBPF 指令；eBPF 返回的 32 字节结果必须与 `OP_KEY` 计算的期望 key 完全相等。与此同时，外壳还独立统计 chart 的滚动值、25 位覆盖 mask、五条 lane 累计值和三类 kind 累计值，防止只伪造 proof。

最终先把 chart 规范化为 tick、lane、delta、kind、flick 和两个浮点字段，连同统计值计算 `cpp_key`，再派生：

$$
K_{\text{final}}=
\operatorname{sekai\_hash}(
\texttt{cpp-final:},\operatorname{cpp\_key}(\text{chart}),
K_{\text{bpf}},\text{chart}).
$$

流的第 $i$ 块为
$\operatorname{sekai\_hash}(\texttt{stream:},K_{\text{final}},\operatorname{u32le}(i))$。
与 C++ blob 异或后即可还原 flag。

最终得到：

```text
SEKAI{eBPF_my_B3l0v3d}
```

## 方法总结

关键是先把载体格式、内层指令、eBPF 一致性校验和 C++ 最终层拆开。四字节 VM 的 `NOTE` 操作已经把每个音符限制到很小的合法域，递归枚举加状态剪枝比搭建通用 VM 反编译器更直接；但只解出 chart 仍不够，还应重建 calldata，让真实 eBPF 返回值与内层 `KEY` 一致，并通过 C++ chart 统计后再解密。
