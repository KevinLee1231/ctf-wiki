---
type: family
tags: [reverse, family, go, rust, jvm, cpp, language-runtime]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/go-rust-jvm-and-cpp-reversing.md
  - ../raw/reverse/HGAME2026-衔尾蛇-wp.md
  - ../raw/reverse/HGAME2026-pvz-wp.md
  - ../raw/reverse/ACTF2026-calc-my-point-wp.md
  - ../raw/reverse/D3CTF2022-d3thon-wp.md
updated: 2026-07-28
---

# Go, Rust, JVM and C++ Reversing

## 作用边界

本页是语言运行时 family，覆盖 Go、Rust、Swift/Kotlin/JVM、D、Haskell、C++、Nuitka/Cython 等编译产物中由运行时对象、符号、类型、异常/协程模型或标准库布局造成的逆向障碍。它不再作为单一 technique。

## 识别信号

- 二进制或字节码中出现语言 runtime 特征：Go `gopclntab`、Rust panic/mangling、JVM class/JAR、C++ RTTI/vtable、Swift/Kotlin/D/Haskell runtime。
- 伪代码主要困难来自对象布局、字符串/slice/interface、异常/协程、反射、动态加载、模板/STL 或大整数库，而不是普通控制流。
- 关键校验入口藏在运行时回调、业务类、native extension、泛型/trait 虚调用、反射方法名或标准库封装后。
- raw 证据显示需要先恢复语言语义，再把提取出的约束、cipher 参数或比较表交给其它 technique。

## 最小证据

- 记录语言/编译器版本线索、符号恢复方式、入口函数、关键类/方法/对象和可观察的输入输出边界。
- 对 Go/Rust/C++，先确认字符串、slice、vector、vtable、interface 或大整数对象的真实内存布局。
- 对 JVM/Kotlin/Spring Boot，至少定位 main/controller/game loop、动态加载点、反编译不可靠位置和真实校验类。
- 对 Cython/Nuitka/native extension，明确 Python stub 与 native 校验函数的调用边界，再决定是否转 Python 字节码或 native 分析页。

## 首轮路由

| 信号 | 先确认 | 下一跳 |
|---|---|---|
| Go binary、`gopclntab`、goroutine/channel、string/slice/interface | 版本、符号恢复、runtime 类型布局和并发边界 | [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md) |
| Rust mangling、panic string、`Option`/`Result`、iterator fusion | 符号 demangle、类型 layout、panic 路径和实际数值约束 | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |
| JVM/Kotlin/Spring Boot/JAR/Android server bytecode | class/jar 入口、反编译是否可靠、协程/state machine、动态加载或隐藏 VM | [python-bytecode-esolangs-and-uefi.md](python-bytecode-esolangs-and-uefi.md)、[vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md) |
| C++ vtable/RTTI/template/STL | 类型恢复、虚调用、对象布局和真实业务函数边界 | [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md) |
| D/Haskell/Swift/Kotlin Native | 语言特定符号、运行时库和中间表示是否比普通伪代码更有信息量 | [mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md) |
| Cython/Nuitka/Python native extension | Python 调用边界和 native 校验函数 | [cython-and-python-extension-checker-recovery.md](cython-and-python-extension-checker-recovery.md)、[python-bytecode-esolangs-and-uefi.md](python-bytecode-esolangs-and-uefi.md) |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| Go/Rust/JVM/.NET metadata、类型和字节码比裸汇编更可靠 | [managed-runtime-metadata-and-bytecode-recovery.md](managed-runtime-metadata-and-bytecode-recovery.md) |
| 比较 API/指令可直接观察目标与输入差异 | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |
| 嵌入 Python/PYD 承载真实校验逻辑 | [cython-and-python-extension-checker-recovery.md](cython-and-python-extension-checker-recovery.md) |

## 合并与拆分结论

- 保留为 family：该页的共同价值是识别语言运行时和对象布局，不是某个单点算法。
- 不拆 Go/Rust/JVM 独立页：当前 WP 案例仍需要在同一层做语言分流，后续某语言积累多个稳定技巧时再拆 technique。
- 不合并进基础工具页：基础工具页说明工具怎么用；本页说明语言运行时导致哪些伪代码和数据结构误判。

## 常见误判

- Go/Rust 字符串和 slice 当成 C 字符串处理，忽略长度字段。
- JVM/Kotlin 只相信反编译 Java，漏掉动态类加载、协程状态机或隐藏 VM。
- C++ 只追函数名，不恢复 vtable/RTTI/对象生命周期。
- Cython/Nuitka 只看 Python stub，没有跟到 native extension 的真实校验。

## 关联页面

- [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md)
- [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md)
- [python-bytecode-esolangs-and-uefi.md](python-bytecode-esolangs-and-uefi.md)
- [cython-and-python-extension-checker-recovery.md](cython-and-python-extension-checker-recovery.md)
- [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md)
- [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [HGAME2026-衔尾蛇-wp](../raw/reverse/HGAME2026-衔尾蛇-wp.md) | Spring Boot/JAR 题要先区分表面业务实现和运行时替换后的真实校验引擎。 |
| [HGAME2026-pvz-wp](../raw/reverse/HGAME2026-pvz-wp.md) | Java 游戏题可以把资源提示、业务类和小范围 hash 状态爆破结合，而不是只搜字符串。 |
| [ACTF2026-calc-my-point-wp](../raw/reverse/ACTF2026-calc-my-point-wp.md) | Rust/GMP/大整数表达式题先还原语言运行时对象和数值约束，再做 CRT 或代数求解。 |
| [D3CTF2022-d3thon-wp](../raw/reverse/D3CTF2022-d3thon-wp.md) | Cython/Python 扩展题应恢复 Python 调用边界和 native 校验函数，不要只看反编译 Python。 |
| [ACTF2026-flagchecker-wp](../raw/reverse/ACTF2026-flagchecker-wp.md) | LoongArch64 Go 静态程序破坏符号恢复，通过反射派生真实方法名；再把 shellcode SM4 层和 8 段 Feistel 环分开求逆。 |
| [0xGame2022-week2-re2-wp](../raw/reverse/0xGame2022-week2-re2-wp.md) | Go `math/big` 调用链是 RSA 校验；重点是提取 `n/c/e` 并注意 `SetString` 的 base 参数，密文是八进制。 |
| [LilacCTF2026-c-plus-plus-plus-plus-wp](../raw/reverse/LilacCTF2026-c-plus-plus-plus-plus-wp.md) | C# Native AOT 中 `XEngine` 是 Twofish-like 16 轮 Feistel；先按 RS/MDS、40 个 round key 和 whitening 恢复固定 key/IV。 |
| [0xGame2024-week4-JustSoSo-wp](../raw/reverse/0xGame2024-week4-JustSoSo-wp.md) | Android 逆向遇到 `native` 方法时，需要沿 `System.loadLibrary` 找到各 ABI 下的 `.so`，按 JNI 导出名定位实现，再把 native 结果带回 Java 数据流。本题最容易写错的是 S 盒初值：源码使用 `256-i`，首项 256 会一直以整数参与置换，只有生成密钥流字节时才截断，不能擅自改成标准 RC4 的 `0..255` 或 `255..0`。 |
| [RCTF2025-fakevdi-wp](../raw/reverse/RCTF2025-fakevdi-wp.md) | 本题的四个环节缺一不可：C# 反混淆找隐藏配置与本地端口、Rust 逆向恢复 AES 登录分支、协议分析得到会话 RC4、按服务端路径重建字节级一致的 BMP。最容易误判的是把缺图当作附件损坏，或只凭视觉制作近似图片；客户端使用整张 BMP 的 MD5 作为下一层 RC4 密钥，所以文件编码、像素布局和元数据同样属于校验输入。 |
| [UMDCTF2019-gosh-wp](../raw/reverse/UMDCTF2019-gosh-wp.md) | Go 二进制即便 strip，`.gopclntab` 仍是恢复函数边界和名称的重要线索。自定义哈希不一定单向：若只由可逆的奇数乘法、取反和 xor-shift 组成，就能在固定宽度整数环上逐步逆转。求出候选后还应重新执行正向函数和原程序双重验证。 |
| [UMDCTF2022-go-away-wp](../raw/reverse/UMDCTF2022-go-away-wp.md) | 跨架构和加密代码不一定是决定性障碍。对 Go、Rust 等包含丰富构建元数据的二进制，第一轮应先做 `file`、`strings` 和路径检索。发现多个候选 flag 时，不能看到 `UMDCTF{...}` 就停止；应结合执行路径、官方答案和产物来源判断哪些是诱饵，哪些是构建时意外泄露。 |
| [WMCTF2023-gohunt-wp](../raw/reverse/WMCTF2023-gohunt-wp.md) | 从二维码密文出发，结合符号信息逆向自定义 base58、异构变换和 xxtea 加密链；tinygo 程序保留符号但字符串全被编码时，应先从特征字符串和外部载体入手，定位编码函数再向上回溯调用链。 |
| [WMCTF2024-rustdroid-wp](../raw/reverse/WMCTF2024-rustdroid-wp.md) | 先 RC4 反推中间密文，再爆破单字节变换比直接逆旋转更简单；长度 43 表明正文部分正好 36 字节：`len("WMCTF{}")=7`。 |

## 原始资料

- [go-rust-jvm-and-cpp-reversing.md](../raw/reverse/go-rust-jvm-and-cpp-reversing.md)
- [HGAME2026-衔尾蛇-wp](../raw/reverse/HGAME2026-衔尾蛇-wp.md)
- [HGAME2026-pvz-wp](../raw/reverse/HGAME2026-pvz-wp.md)
- [ACTF2026-calc-my-point-wp](../raw/reverse/ACTF2026-calc-my-point-wp.md)
- [D3CTF2022-d3thon-wp](../raw/reverse/D3CTF2022-d3thon-wp.md)
