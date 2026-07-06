---
type: family
tags: [misc, pwn, family, vm, z3, sandbox, game, constraints]
skills: [ctf-misc, ctf-pwn]
raw:
  - ../raw/misc/vm-z3-sandbox-and-game-basics.md
  - ../raw/pwn/WMCTF2025-wm-eat-some-qanux-wp.md
  - ../raw/pwn/D3CTF2019-babyrop-wp.md
updated: 2026-07-06
---

# VM, Z3, Sandbox and Game Basics

## 作用边界

本页是 Misc 中“规则系统” family，用于处理小型 VM/解释器、WASM/Roblox/游戏资源、PyInstaller/marshal、Z3/约束求解、Kubernetes/RBAC、浮点精度、Lua/Ruby/Python sandbox 和自定义汇编器过滤等题型。

它只承担首轮分流：如果主要任务是逆向复杂 VM 或字节码，转 Reverse family；如果已经形成内存破坏 primitive，转 Pwn；如果是真实内网/容器提权，转 Pentest。

## 识别信号

- 题目给出游戏、解释器、字节码、规则表、约束网络、WASM、Roblox place、PyInstaller 包、marshal code、K8s manifest、浮点交易系统或自定义 sandbox。
- 关键不是文件 carving，而是把规则还原成可执行模型、patch、约束或状态转移。
- 可通过小脚本、Z3、模拟器、patch 或重放交互快速验证候选路线。

## 最小证据

- 明确规则载体：WASM、Python bytecode、marshal、游戏资源、逻辑门网络、YARA 条件、K8s API、浮点计算、VM opcode 或 sandbox 语言。
- 定义输入、状态、转移和胜利条件；不要只描述“像游戏”或“像 VM”。
- 对 patch 路线，先确认校验点和可改字段；对 Z3 路线，先确认变量域和约束规模；对 sandbox 路线，先确认可用 API 和过滤阶段。
- 如果出现读写/执行 primitive，要说明是否已经超出 Misc，是否应 pivot 到 Pwn/Reverse/Pentest。

## 首轮路由

| 证据形态 | 先做什么 | 下一跳 |
|---|---|---|
| WASM/Roblox/游戏 AI 可 patch | 先找评分、胜利条件、资源加载和客户端状态，patch 后用最小交互验证。 | [game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md) |
| PyInstaller/marshal/opcode remap | 先解包、确定 Python 版本和 opcode 表，再反汇编/反编译；复杂 VM 转 Reverse。 | [python-bytecode-esolangs-and-uefi.md](python-bytecode-esolangs-and-uefi.md) |
| Z3/YARA/逻辑门/类型系统 | 先抽变量域和约束，不要直接暴力；若状态递推或 oracle 更强，转 oracle 页面。 | [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md) |
| K8s/RBAC/容器 API | 先枚举 service account、token、verb/resource、namespace 和可创建对象；成功后转 Pentest。 | [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md) |
| 浮点精度/经济系统 | 先找舍入方向、整数转换、库存/余额同步点和可重复交易循环。 | [interpreter-jit-canary-and-integer-exploits.md](interpreter-jit-canary-and-integer-exploits.md) |
| Lua/Ruby/Python sandbox | 先确定语言运行时和过滤阶段；Python 走 pyjail，Lua/Ruby 看 TracePoint/函数名/元表等隐式执行点。 | [pyjails.md](pyjails.md) |
| 自定义 VM opcode 或 syscall 被过滤 | 判断过滤发生在汇编器、loader 还是解释器；若已有 SP/RIP/内存 primitive，转 Pwn。 | [oob-jit-parser-primitives-family.md](oob-jit-parser-primitives-family.md) |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 | 对应路线 |
|---|---|---|
| [WMCTF2025-wm-eat-some-qanux-wp](../raw/pwn/WMCTF2025-wm-eat-some-qanux-wp.md) | 自定义汇编器会把明文 `svc` 替换成 NOP，但解释器内部仍实现 syscall opcode；SP 越界可直接进入内部 opcode 路线。 | VM 语义差异 + Pwn primitive |
| [D3CTF2019-babyrop-wp](../raw/pwn/D3CTF2019-babyrop-wp.md) | VM 指令语义恢复后发现 guest stack 操作能写宿主返回地址；这类题应先建 opcode 到宿主上下文的映射，再转 Pwn primitive。 | VM 语义差异 + 返回地址覆盖 |

## 合并与拆分结论

- 保留为 `family`：raw 横跨游戏 patch、Python 包/字节码、约束求解、K8s、浮点、sandbox 和 VM opcode，单一 technique 无法准确概括。
- 不并入 [misc-cross-category-triage-family.md](misc-cross-category-triage-family.md)：总 triage 只决定是否进入本页，本页负责规则系统内部二级分流。
- 不拆 Z3、WASM、K8s 等小页：当前 raw 多为短案例集合，拆分后容易形成弱页；后续高频路线再独立成 technique。

## 常见陷阱

- 看到 VM/bytecode 就直接用 Reverse 工具，没先确认是否只是小规则系统或可 patch 游戏。
- 用 Z3 前没有约束变量域，导致 solver 模型不对应实际输入格式。
- K8s 题只看 pod 内文件，没枚举 service account 权限和 API verbs。
- 浮点题用十进制直觉推导，未复现实际语言的二进制浮点和整数转换。
- 自定义汇编过滤只尝试绕文本，忽略解释器内部 opcode/helper 可能直接可用。

## 关联技巧

- [misc-cross-category-triage-family.md](misc-cross-category-triage-family.md)
- [game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md)
- [python-bytecode-esolangs-and-uefi.md](python-bytecode-esolangs-and-uefi.md)
- [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md)
- [pyjails.md](pyjails.md)
- [oob-jit-parser-primitives-family.md](oob-jit-parser-primitives-family.md)
- [interpreter-jit-canary-and-integer-exploits.md](interpreter-jit-canary-and-integer-exploits.md)
- [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md)

## 原始资料

- [vm-z3-sandbox-and-game-basics.md](../raw/misc/vm-z3-sandbox-and-game-basics.md)
- [WMCTF2025-wm-eat-some-qanux-wp](../raw/pwn/WMCTF2025-wm-eat-some-qanux-wp.md)
- [D3CTF2019-babyrop-wp](../raw/pwn/D3CTF2019-babyrop-wp.md)
