---
type: family
tags: [cross-category, family, interactive, container, jail, solver, oracle]
skills: [ctf-solve-challenge, ctf-pwn, ctf-pentest, ctf-cloud-infra, ctf-reverse]
raw:
  - ../raw/pwn/interactive-containers-jails-and-solvers.md
updated: 2026-06-12
---

# Interactive Containers, Jails and Solvers

## 作用边界

本页是交互环境、容器边界、jail 变体和小型求解器的跨方向 family 页。它处理的不是同一个技巧，而是一类“题目给了可交互运行环境或规则系统，必须先判断可借用的边界在哪里”的问题。

如果证据已经明确是 Python jail、bash/rbash、纯编码链、图像隐写或 WebSocket 游戏，应转入更具体页面。本页保留的价值是把容器、交互 oracle、游戏规则、运行时 helper、虚拟化边界和受限执行环境的首轮判断串起来。

## 识别信号

- 题目提供远程交互服务、容器、Docker/BuildKit、rvim、受限命令环境、模拟器、ROM 切换、marshal 代码载入或多阶段小游戏。
- 静态附件不足以出 flag，需要维护会话状态、并发连接、socket oracle、交互脚本或求解器。
- 边界可能是宿主机挂载、Docker socket、capability、fd trick、运行时 helper、虚拟机状态或受限语言的 escape。
- 输出常表现为 partial oracle、频率分布、拼图约束、二维码网格、图像碎片或需要策略博弈的状态。

## 最小证据

- 记录交互协议：输入输出顺序、token、nonce、会话状态、超时、连接数和是否可并发。
- 明确环境边界：容器权限、挂载、socket、capability、fd、可执行文件、helper 进程和网络访问。
- 对规则系统给出最小模型：状态表示、转移规则、胜负条件、约束规模或 oracle 反馈。
- 能判断下一步是 jail escape、容器逃逸、solver、pwn primitive、forensics 重组还是 crypto oracle。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| 多阶段交互、token 先提交、状态后揭示 | 先还原协议顺序和不可变承诺，再判断是否是 oracle relay、博弈求解或 crypto 约束 | [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md), [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md) |
| Docker/BuildKit、privileged、Docker socket、CAP_SYS_ADMIN | 先枚举挂载、socket、capability 和 worker，再判断信息泄露、secret 读取或容器逃逸 | [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md), [linux-privesc.md](linux-privesc.md) |
| rvim、bash/rbash、受限 shell 或命令白名单 | 先识别 eval/quoting/字符集/环境变量，再转 shell jail 页面 | [bashjails.md](bashjails.md), [source-backdoors-and-restricted-shell-tricks.md](source-backdoors-and-restricted-shell-tricks.md) |
| Python marshal、代码对象注入、受限 Python helper | 先判断 namespace 和 code object 可控性，再转 Python jail | [pyjails.md](pyjails.md) |
| C 代码 jail、emoji 标识符、可嵌 gadget | 先把字符集和编译产物映射成字节，再判断是否进入 pwn syscall/ROP | [stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md), [seccomp-ret2dlresolve-and-runtime-primitives.md](seccomp-ret2dlresolve-and-runtime-primitives.md) |
| 模拟器 ROM 切换、memfd、运行时 helper | 先保存状态、dump 内存或找到 helper 边界，再转 reverse/forensics | [vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md), [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md) |
| Nonogram、15-puzzle、100 prisoners、Levenshtein oracle | 先建规则模型和验证器，再生成最短 solver | [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md) |
| 像素边缘重组、二维码输出、碎片拼接 | 先恢复几何/网格证据，再转图像或二维码页面 | [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md) |

## 合并与拆分结论

本页应保留为 family。它与 [pyjails.md](pyjails.md) 和 [bashjails.md](bashjails.md) 不重复：后两者处理具体语言/ shell 逃逸，本页处理交互环境、容器边界和规则系统的首轮分流。也不并入 [vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md)，因为容器与交互 oracle 的证据形态不同。

## 常见陷阱

- 只看附件不建交互协议，导致远程状态机和本地复现不一致。
- 容器题只跑 `id`，没查 socket、capability、mount、worker 和 host path。
- 把规则题当爆破，未先判断状态空间是否需要 C++、缓存或数学策略。
- jail escape 后停在 shell，不继续枚举内部服务、flag 进程或网络边界。
- 把 memfd/ROM/helper 当普通文件分析，忽略运行时状态保存。

## 关联技巧

- [cross-category-triage-family.md](cross-category-triage-family.md)
- [bashjails.md](bashjails.md)
- [pyjails.md](pyjails.md)
- [vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md)
- [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md)
- [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md)
- [linux-privesc.md](linux-privesc.md)
- [pwn-tooling.md](pwn-tooling.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [ACTF2026-farthest2026-wp](../raw/pwn/ACTF2026-farthest2026-wp.md) | DOS/dosemu2/COMCOM64 sidecar 形成容器边界逃逸，先确认虚拟化边界、helper 加载和 host 侧执行点。 |

## 原始资料

- [interactive-containers-jails-and-solvers.md](../raw/pwn/interactive-containers-jails-and-solvers.md)
