---
type: family
tags: [reverse, family, patching, oracle, tracing, hook]
skills: [ctf-reverse, ctf-mobile]
raw:
  - ../raw/reverse/runtime-patching-oracles-and-tracing.md
  - ../raw/reverse/WMCTF2025-videoplayer-wp.md
  - ../raw/reverse/WMCTF2025-want2become-magicalgirl-wp.md
updated: 2026-07-27
---

# Runtime Patching, Oracles and Tracing

## 作用边界

本页是运行时 patch、动态 oracle、trace、hook、coredump 和执行流降维 family。它适用于静态伪代码成本高，但可以通过运行时观察、替换数据、插桩或构造 oracle 直接恢复关键状态的题。

它与 [signal-trace-and-packed-anti-analysis.md](signal-trace-and-packed-anti-analysis.md) 分工不同：本页偏主动 patch/hook/oracle，后者偏信号处理器、父子进程、packed module dump 和 trace 反演。

## 识别信号

- 静态分析能定位校验或解密阶段，但完整还原成本高，运行时中间 buffer、key、vector、返回值或比较状态可观察。
- 程序可被 Frida、smali patch、LD_PRELOAD、INT3、coredump、trace、faketime 或动态符号执行降维。
- 只改成功/失败返回值不够，因为后续还复用原始 key、hash、向量或上下文对象。
- Android/Java/native 混合、libart self-hook、Frida 检测或 unstable hook point 需要改用 smali trace/静态 patch。

## 最小证据

- 目标状态是什么：明文 buffer、key、hash/vector、分支条件、比较结果、oracle bit、VM 指令或解密数据。
- 选择的观察点在数据被使用之前，而不是只在最终返回后。
- Patch 或 hook 不改变后续必要数据；如果会改变，要同步替换上下文对象。
- 能用正向 check 验证恢复结果，而不只依赖“程序显示成功”。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| 只 patch 返回值会登录但解密失败 | 后续是否复用 key/hash/vector | patch 数据本身或 dump 正确上下文 |
| Frida/hook 被检测 | 检测点、self-hook、native 层和 Java 层谁改变语义 | smali trace、静态 patch 或早期 attach |
| INT3/coredump oracle | 崩溃点是否携带候选状态，coredump 是否稳定生成 | 自动化候选输入和 dump 解析 |
| timing side-channel | 响应时间是否和比较进度/分支相关 | 重复采样、降噪、固定环境 |
| LD_PRELOAD oracle | 动态链接符号可拦截，目标未静态链接/校验 libc | 劫持函数返回或记录参数 |
| VM printf/trace 到 Z3 | 输出/trace 可转成约束，而非完整手工反编译 | [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md) |
| 多线程/反调试干扰 | decoy thread、signal handler、anti-debug 是否影响观察点 | [anti-analysis.md](anti-analysis.md) |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2025-videoplayer-wp](../raw/reverse/WMCTF2025-videoplayer-wp.md) | 只把机器校验函数返回值改成 1 会登录成功但解密失败，因为后续还使用 MD5 vector；必须把用于解密的 MD5 数据本身替换成目标机器值。 |
| [WMCTF2025-want2become-magicalgirl-wp](../raw/reverse/WMCTF2025-want2become-magicalgirl-wp.md) | Frida 检测和 libart self-hook 干扰动态 hook 时，smali trace 能确认 Java 层魔改 XXTEA 的真实移位方向和轮数。 |
| [HGAME2026-steins-gate-wp](../raw/pwn/HGAME2026-steins-gate-wp.md) | 逐字节比较错误会在不同位置 panic，PIE 仍保留稳定页内低 12 位偏移；用崩溃地址作为比较进度 oracle 爆破输入。 |
| [西湖论剑2023-BabyRE-wp](../raw/reverse/西湖论剑2023-BabyRE-wp.md) | 主函数只做自定义 base8 第一层，第二层藏在 `atexit` 回调；用输入尾部 6 位十进制 RC4 key 约束反解。 |
| [HGAME2026-marionette-wp](../raw/reverse/HGAME2026-marionette-wp.md) | 父进程用 `ptrace` 调度子进程 `int3; ret` block；hook 记录 RIP trace 后，还原输入差分和 AES-NI 校验。 |
| [LilacCTF2026-ezpython-wp](../raw/reverse/LilacCTF2026-ezpython-wp.md) | PyInstaller runtime hook 把自定义 `a85decode/b64decode` 写入 `builtins`，并动态改 `MX.__code__` 后才是 XXTEA 真实轮函数。 |
| [NCTF2026-hook-my-secret-wp](../raw/reverse/NCTF2026-hook-my-secret-wp.md) | 运行时 hook、patch 或 oracle 可观测关键状态，先选断点和最小输入。 |
| [NCTF2026-nomybank-wp](../raw/reverse/NCTF2026-nomybank-wp.md) | Godot PCK 密钥、运行时解密 DLL、TLS callback hook 和 SMC 共同隐藏真实校验；先恢复资源和动态补丁链。 |
| [SUCTF2026-protocolWP](../raw/reverse/SUCTF2026-protocolWP.md) | HTTP 路由很薄，body 先 hex 再进私有协议帧；区分格式错、比较失败和 block 变换后再反推 payload。 |
| [SUCTF2026-WestWP](../raw/reverse/SUCTF2026-WestWP.md) | 81 轮 permutation + dispatch table 更新共享状态；逆三个 rotate/add/xor helper 后，用 Unicorn 推进状态并约束求输入。 |
| [0xGame2025-week3-Calamity-Fortune-wp](../raw/reverse/0xGame2025-week3-Calamity-Fortune-wp.md) | 本题用“猜错也给 flag”的表象隐藏伪答案，再借构造函数中的 inline hook 把正确分支转到另一套校验。分析 Windows 程序时，若同时看到 `VirtualProtect`、`FlushInstructionCache` 和 `mov rax, imm64; jmp rax`，应优先检查运行时代码改写，而不能只沿 `main()` 的静态控制流追踪。 |
| [0xGame2025-week4-幻视调律-wp](../raw/reverse/0xGame2025-week4-幻视调律-wp.md) | 本题用“可完整解出的假 flag”诱导选手过早结束静态分析。以后遇到结果主动声明自己是 fake、程序导入的比较函数行为与返回结果不一致，或调试时调用目标跳入非模块代码区，应继续核对运行时调用目标。算法逆向时还要特别留意状态复用和重叠分组：本题的 S 盒会跨调用保留，而 11 个字组成的 10 个相邻字对必须逆序解密，忽略任一细节都会得到看似合理但错误的输出。 |
| [ACTF2023-tree-wp](../raw/reverse/ACTF2023-tree-wp.md) | 本题的决定性障碍是恢复 Clang ASTMatcher，而不是普通控制流或字符串校验。分析时应从全局 matcher 构造函数中的节点类型、操作符字符串、绑定 ID 和回调逻辑入手；必要时编写最小 ASTMatcher 参考程序辅助 BinDiff。动态调试则适合直接观察三个全局计数，将复杂 matcher 拆成“循环数量、二元运算计分、类继承结构”三个可独立验证的条件。最终输入只需在 AST 层满足约束，源码本身无需执行。 |
| [UMDCTF2023-clutter-wp](../raw/reverse/UMDCTF2023-clutter-wp.md) | 理解 VeSP 的加载、加法和内存写入语义，从 verbose 轨迹按执行顺序提取低字节，而不是按随机内存地址排序；程序无输出指令，却频繁把小整数写入一段特定内存范围时，模拟器 trace 往往就是观察通道。 |
| [UMDCTF2025-ls-wp](../raw/reverse/UMDCTF2025-ls-wp.md) | `ptrace` 在 syscall entry 修改 `orig_rax`，把输入字符变成系统调用号的一次性 XOR 密钥。应结合寄存器参数、返回值用途和后续检查恢复“表面编号 → 目标编号 → 字符”映射，不能只相信源码函数名。 |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| API/比较/阶段边界适合 hook、trace 与状态快照 | [trace-hook-and-state-snapshot-reconstruction.md](trace-hook-and-state-snapshot-reconstruction.md) |
| anti-debug/self-check/environment probe 先阻断真实路径 | [anti-debug-self-check-and-environment-bypass.md](anti-debug-self-check-and-environment-bypass.md) |
| 比较点可直接观察候选与目标明文 | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |

## 合并与拆分结论

- 保留为 family：运行时 patch/oracle/trace 是多种手段的路线族，不是单一技术。
- 不合并进 `anti-analysis.md`：反分析页解决“如何跑起来”，本页解决“跑起来后如何观测和降维”。
- 不合并进 `signal-trace-and-packed-anti-analysis.md`：信号处理器和 packed dump 的证据形态更具体，仍需独立二级页。

## 常见误判

- Patch 返回值后就认为解题完成，忽略后续数据依赖。
- Hook 点太晚，只能看到真假结果，看不到恢复所需中间状态。
- 动态 trace 引入检测副作用，导致观察到的是反调试分支。
- Timing oracle 没有重复采样，网络或调度噪声被当成有效 bit。

## 关联页面

- [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md)
- [signal-trace-and-packed-anti-analysis.md](signal-trace-and-packed-anti-analysis.md)
- [anti-analysis.md](anti-analysis.md)
- [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)
- [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md)
- [frida-angr-lldb-and-x64dbg.md](frida-angr-lldb-and-x64dbg.md)

## 原始资料

- [runtime-patching-oracles-and-tracing.md](../raw/reverse/runtime-patching-oracles-and-tracing.md)
- [WMCTF2025-videoplayer-wp](../raw/reverse/WMCTF2025-videoplayer-wp.md)
- [WMCTF2025-want2become-magicalgirl-wp](../raw/reverse/WMCTF2025-want2become-magicalgirl-wp.md)
