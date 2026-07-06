---
type: family
tags: [reverse, family, decryption, constraints, strings, lattice]
skills: [ctf-reverse, ctf-crypto]
raw:
  - ../raw/reverse/self-decrypting-strings-and-lattice-patterns.md
  - ../raw/reverse/WMCTF2025-appfriend-wp.md
  - ../raw/reverse/WMCTF2025-catfriend-wp.md
  - ../raw/reverse/WMCTF2025-want2become-magicalgirl-wp.md
  - ../raw/reverse/Bugku-week2_re1-wp.md
updated: 2026-07-06
---

# Self-Decrypting, String and Lattice Patterns

## 作用边界

本页是 Reverse 中“把隐藏校验恢复成可计算模型”的 family 页，覆盖多层自解密、嵌入式归档、XOR/stack string、prefix hash 爆破、CVP/LLL、决策树函数、GF(2^8) 线性恢复、ROP 链混淆和魔改常见密码算法。

它不是一个具体 technique。首轮要判断隐藏逻辑是“需要 dump 每一层”、还是“提取常量后直接解码”、还是“转成约束/线性代数/格问题”、还是“恢复魔改 cipher 的真实轮函数”。不同判断会决定是否进入调试自动化、crypto、solver 或普通脚本解密。

## 识别信号

- 静态字符串不完整，运行时才解密或生成新的 ELF/ZIP/代码段/校验函数。
- `.rodata`、stack string、资源段、native so、Flutter/Java 层或 VM 中藏有 key、S-box、密文、线性矩阵或 hash 前缀。
- 输入校验不是单一比较，而是多函数决策树、矩阵方程、模约束、格最近向量、魔改 AES/SM4/ChaCha/XXTEA。
- 需要把动态行为降维成可重复的解密脚本、约束求解器或 forward checker。

## 最小证据

- 已定位隐藏数据来源：段、偏移、资源、运行时 dump、函数返回值或 trace 输出。
- 能说明数据变换类型：XOR、流/分组密码、hash 前缀、线性方程、格约束、决策树或多层自解密。
- 能提取最小常量集合：key、密文、S-box、矩阵、模数、比较常量、轮数或函数调用顺序。
- 能用脚本复算至少一个中间结果，证明模型与程序行为一致。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| 多层自解密、运行时生成新代码或新二进制 | 先自动化 dump/patch/trace，确认每层入口和输出，不要手工逐层点开 | [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md), [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md) |
| 嵌入 ZIP、资源段、XOR license、stack string | 先按偏移和符号边界提取数据，再复算解密脚本 | [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md) |
| 静态数组、短汇编片段、原位字节变换 | 先按真实遍历方向模拟指令语义，确认旧值/新值依赖和 `uint8/uint32` 截断 | [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md) |
| prefix hash、MD5 前缀、短域爆破 | 先估计搜索空间和剪枝条件，保留程序一致的 hash/编码顺序 | [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md) |
| CVP/LLL、整数范围约束、模线性组合 | 先把校验表达式转成矩阵/格模型，并确认变量范围和模数 | [lattice-and-lwe.md](lattice-and-lwe.md), [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| GF(2^8)、GF(2) 或线性字节方程 | 先确认域乘法、多项式和矩阵方向，再做高斯消元 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| 决策树函数、海量小函数比较 | 先批量提取常量和路径条件，再生成 forward checker 或 solver | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |
| 发现 AES/SM4/ChaCha/XXTEA 常量但流程异常 | 先恢复真实轮函数、轮数、S-box、移位方向和加解密同构性 | [embedded-python-pyd-custom-aes.md](embedded-python-pyd-custom-aes.md), [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md) |
| ROP 链或控制流本身被当作校验 | 先 trace gadget 序列和寄存器状态，再反向组装可计算模型 | [anti-analysis.md](anti-analysis.md), [stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md) |

## 合并与拆分结论

本页应保留为 family。自解密、字符串恢复、格约束、GF 线性恢复和魔改 cipher 的第一步证据不同，不能合并成单一 technique；但它们都服务于“把隐藏校验恢复成可复算模型”，作为 Reverse 首轮后的二级入口有价值。

当前不重命名文件：现有 slug 虽然较长，但能覆盖页面中的三条主轴，且已有多处链接。若后续拆出专门页面，可再把本页收敛为 `reverse-decryption-and-constraint-recovery.md`。

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2025-appfriend-wp](../raw/reverse/WMCTF2025-appfriend-wp.md) | native so 中识别 SM4 后，关键是提取 16 字节 key 和 48 字节密文做可复算解密，而不是只描述 native 加密。 |
| [WMCTF2025-catfriend-wp](../raw/reverse/WMCTF2025-catfriend-wp.md) | 魔改 ChaCha20 仍是流密码，恢复轮数、QR 函数和 VM 实现的 xor 后，加密流程可直接复用为解密流程。 |
| [WMCTF2025-want2become-magicalgirl-wp](../raw/reverse/WMCTF2025-want2become-magicalgirl-wp.md) | Flutter 层魔改 AES 调整 S-box 和 AddRoundKey/MixColumns 顺序，Java 层魔改 XXTEA 改移位方向和轮数；恢复脚本必须按真实执行流逆序。 |
| [Bugku-week2_re1-wp](../raw/reverse/Bugku-week2_re1-wp.md) | 只有汇编文本和 `enc[]` 时，关键是按倒序循环模拟原位数组变换；`input[i + 1]` 已经是新值，遍历方向比反汇编工具选择更重要。 |

## 常见陷阱

- 只看到 AES/SM4/ChaCha 常量就套标准库，忽略轮函数、S-box、轮数或操作顺序已被改动。
- 提取数据时没有确认边界，把相邻符号或 padding 当密文。
- 把数学求解写成一次性脚本，但没有用程序中间结果做 forward check。
- LLL/CVP、GF 和普通整数线性代数混用，导致模型看似能解但回代失败。
- 多层自解密只 dump 最后一层，遗漏了前面阶段生成的 key 或修补逻辑。

## 关联技巧

- [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md)
- [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)
- [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)
- [embedded-python-pyd-custom-aes.md](embedded-python-pyd-custom-aes.md)
- [lattice-and-lwe.md](lattice-and-lwe.md)
- [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md)
- [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md)
- [reverse-tooling.md](reverse-tooling.md)

## 原始资料

- [self-decrypting-strings-and-lattice-patterns.md](../raw/reverse/self-decrypting-strings-and-lattice-patterns.md)
- [WMCTF2025-appfriend-wp](../raw/reverse/WMCTF2025-appfriend-wp.md)
- [WMCTF2025-catfriend-wp](../raw/reverse/WMCTF2025-catfriend-wp.md)
- [WMCTF2025-want2become-magicalgirl-wp](../raw/reverse/WMCTF2025-want2become-magicalgirl-wp.md)
- [Bugku-week2_re1-wp](../raw/reverse/Bugku-week2_re1-wp.md)
