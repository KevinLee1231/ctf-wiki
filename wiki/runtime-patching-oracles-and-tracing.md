---
type: family
tags: [reverse, family, patching, oracle, tracing, hook]
skills: [ctf-reverse, ctf-mobile]
raw:
  - ../raw/reverse/runtime-patching-oracles-and-tracing.md
  - ../raw/reverse/WMCTF2025-videoplayer-wp.md
  - ../raw/reverse/WMCTF2025-want2become-magicalgirl-wp.md
updated: 2026-07-11
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
| [Bugku-BabyRE-wp](../raw/reverse/Bugku-BabyRE-wp.md) | 主函数只做自定义 base8 第一层，第二层藏在 `atexit` 回调；用输入尾部 6 位十进制 RC4 key 约束反解。 |
| [HGAME2026-marionette-wp](../raw/reverse/HGAME2026-marionette-wp.md) | 父进程用 `ptrace` 调度子进程 `int3; ret` block；hook 记录 RIP trace 后，还原输入差分和 AES-NI 校验。 |
| [LilacCTF2026-ezpython-wp](../raw/reverse/LilacCTF2026-ezpython-wp.md) | PyInstaller runtime hook 把自定义 `a85decode/b64decode` 写入 `builtins`，并动态改 `MX.__code__` 后才是 XXTEA 真实轮函数。 |
| [NCTF2026-hook-my-secret-wp](../raw/reverse/NCTF2026-hook-my-secret-wp.md) | 运行时 hook、patch 或 oracle 可观测关键状态，先选断点和最小输入。 |
| [NCTF2026-nomybank-wp](../raw/reverse/NCTF2026-nomybank-wp.md) | Godot PCK 密钥、运行时解密 DLL、TLS callback hook 和 SMC 共同隐藏真实校验；先恢复资源和动态补丁链。 |
| [SUCTF2026-protocolWP](../raw/reverse/SUCTF2026-protocolWP.md) | HTTP 路由很薄，body 先 hex 再进私有协议帧；区分格式错、比较失败和 block 变换后再反推 payload。 |
| [SUCTF2026-WestWP](../raw/reverse/SUCTF2026-WestWP.md) | 81 轮 permutation + dispatch table 更新共享状态；逆三个 rotate/add/xor helper 后，用 Unicorn 推进状态并约束求输入。 |

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
