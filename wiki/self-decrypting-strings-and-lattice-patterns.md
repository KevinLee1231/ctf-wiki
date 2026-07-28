---
type: family
tags: [reverse, family, decryption, constraints, strings, lattice]
skills: [ctf-reverse, ctf-crypto]
raw:
  - ../raw/reverse/self-decrypting-strings-and-lattice-patterns.md
  - ../raw/reverse/WMCTF2025-appfriend-wp.md
  - ../raw/reverse/WMCTF2025-catfriend-wp.md
  - ../raw/reverse/WMCTF2025-want2become-magicalgirl-wp.md
updated: 2026-07-28
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
| 发现 AES/SM4/ChaCha/XXTEA 常量但流程异常 | 先恢复真实轮函数、轮数、S-box、移位方向和加解密同构性 | [block-mode-misuse-family.md](block-mode-misuse-family.md), [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md) |
| ROP 链或控制流本身被当作校验 | 先 trace gadget 序列和寄存器状态，再反向组装可计算模型 | [anti-analysis.md](anti-analysis.md), [stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md) |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| 字符串/状态只在运行时解密并短暂存在 | [trace-hook-and-state-snapshot-reconstruction.md](trace-hook-and-state-snapshot-reconstruction.md) |
| 解密后仍是 Base/字符集/码表/位序表示链 | [layered-encoding-and-symbol-mapping-recovery.md](layered-encoding-and-symbol-mapping-recovery.md) |
| 比较点可直接截获候选或目标明文 | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |

## 合并与拆分结论

本页应保留为 family。自解密、字符串恢复、格约束、GF 线性恢复和魔改 cipher 的第一步证据不同，不能合并成单一 technique；但它们都服务于“把隐藏校验恢复成可复算模型”，作为 Reverse 首轮后的二级入口有价值。

当前不重命名文件：现有 slug 虽然较长，但能覆盖页面中的三条主轴，且已有多处链接。若后续拆出专门页面，可再把本页收敛为 `reverse-decryption-and-constraint-recovery.md`。

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2025-appfriend-wp](../raw/reverse/WMCTF2025-appfriend-wp.md) | native so 中识别 SM4 后，关键是提取 16 字节 key 和 48 字节密文做可复算解密，而不是只描述 native 加密。 |
| [WMCTF2025-catfriend-wp](../raw/reverse/WMCTF2025-catfriend-wp.md) | 魔改 ChaCha20 仍是流密码，恢复轮数、QR 函数和 VM 实现的 xor 后，加密流程可直接复用为解密流程。 |
| [WMCTF2025-want2become-magicalgirl-wp](../raw/reverse/WMCTF2025-want2become-magicalgirl-wp.md) | Flutter 层魔改 AES 调整 S-box 和 AddRoundKey/MixColumns 顺序，Java 层魔改 XXTEA 改移位方向和轮数；恢复脚本必须按真实执行流逆序。 |
| [TamilCTF2021-GoldDigger-wp](../raw/reverse/TamilCTF2021-GoldDigger-wp.md) | 验证函数形如 `input[index[i]] + const == target[i]`；提取置换表和目标数组后直接反填输入。 |
| [强网杯2019-JustRe-wp](../raw/reverse/强网杯2019-JustRe-wp.md) | Flag 分两段：前半段是 DWORD 加法/低字节 XOR 约束，后半段是固定 3DES-ECB 密文和 24 字节 key。 |
| [Xp0intCTF2017-MaybeNotStandrad-wp](../raw/reverse/Xp0intCTF2017-MaybeNotStandrad-wp.md) | 输入 45 字节、输出 60 字符且有 64 字符表，是标准 Base64 结构加非标准字母表；先还原表再解码。 |
| [0xGame2022-week1-re2-wp](../raw/reverse/0xGame2022-week1-re2-wp.md) | 36 字符中间段经过带反馈的 `uint32` 乘加/XOR 哈希，目标数组固定；用 Z3 保留 32 位溢出语义求解。 |
| [D3CTF2021-ancient-wp](../raw/reverse/D3CTF2021-ancient-wp.md) | 算术编码和编译期字符串保护组合，先 dump 固定分布表和目标编码再反解输入。 |
| [D3CTF2021-white-give-wp](../raw/reverse/D3CTF2021-white-give-wp.md) | LLVM pass 全局变量 AES、常数拆分和 MBA 表达式替换，先 dump 明文常量再还原校验流。 |
| [LilacCTF2026-nineapple-wp](../raw/reverse/LilacCTF2026-nineapple-wp.md) | iOS Swift 九宫格手势锁给出 `weight/target_all/map_list`；无需操作 UI，按加权和反查每个字符路径。 |
| [0xGame2025-week4-云消雾散-wp](../raw/reverse/0xGame2025-week4-云消雾散-wp.md) | 看到 RC4 风格的 S 盒更新时，不能默认算法一定会生成密钥流并执行异或；本题真正的数据操作是分组内的位置交换。逆置换必须从末状态倒序撤销，并严格复现循环边界。处理图片时也应以程序的真实文件读写为准：本题固定保留 1078 字节、仅处理 822528 字节，余下 72 字节不变，这些边界条件缺一不可。 |
| [MoeCTF2024-sm4-wp](../raw/reverse/MoeCTF2024-sm4-wp.md) | 标准密码算法题也要检查调用点的内存语义。源码中声明的 key 不一定是运行时 key，数组相邻关系也必须以目标二进制为准；这里一个 NUL off-by-one 改变了 SM4 密钥首字节，解释了使用表面密钥始终解不出的现象。 |
| [MoeCTF2024-XTEA-wp](../raw/reverse/MoeCTF2024-XTEA-wp.md) | 识别 XTEA 后仍要还原调用层的数据流。重叠分组会让第二次加密依赖第一次的中间密文，逆向顺序必须与加密调用顺序相反；若直接把 12 字节拆成两个普通独立块，永远无法得到正确 key。 |

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
- [cython-and-python-extension-checker-recovery.md](cython-and-python-extension-checker-recovery.md)
- [lattice-and-lwe.md](lattice-and-lwe.md)
- [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md)
- [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md)
- [reverse-tooling.md](reverse-tooling.md)

## 原始资料

- [self-decrypting-strings-and-lattice-patterns.md](../raw/reverse/self-decrypting-strings-and-lattice-patterns.md)
- [WMCTF2025-appfriend-wp](../raw/reverse/WMCTF2025-appfriend-wp.md)
- [WMCTF2025-catfriend-wp](../raw/reverse/WMCTF2025-catfriend-wp.md)
- [WMCTF2025-want2become-magicalgirl-wp](../raw/reverse/WMCTF2025-want2become-magicalgirl-wp.md)
