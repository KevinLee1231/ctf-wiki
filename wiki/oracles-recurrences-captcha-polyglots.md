---
type: family
tags: [misc, family, oracle, recurrence, captcha, polyglot]
skills: [ctf-misc, ctf-web, ctf-crypto]
raw:
  - ../raw/misc/oracles-recurrences-captcha-polyglots.md
  - ../raw/crypto/ACTF2026-ohmycaptcha-wp.md
  - ../raw/crypto/HGAME2026-flux-wp.md
  - ../raw/misc/NCTF2026-ezprotocol-wp.md
  - ../raw/misc/SU_MirrorBus9WP.md
updated: 2026-07-06
---

# Oracles, Recurrences, CAPTCHA and Polyglots

## 作用边界

本页是 misc 中 oracle、递推、约束、CAPTCHA 和 polyglot 规则系统 family。它覆盖比较 oracle、timeout oracle、XSLT/DSL VM、JavaScript 数值边界、OEIS/递推、矩阵快速幂、QR 结构约束、动态字体 CAPTCHA、Brainfuck/Piet 多层题和 Levenshtein oracle。

## 识别信号

- 服务或脚本不给明文答案，但比较结果、超时、距离、错误类型或成功/失败能作为 oracle。
- 输入输出满足递推、矩阵幂、序列查询、有限状态机或覆盖约束。
- CAPTCHA/字体/QR/polyglot 不是图像处理本身，而是结构约束或自动化识别问题。
- 题目位于 misc，但证据可能 pivot 到 web SQLi timeout、crypto hash length extension 或 encoding。

## 最小证据

- oracle 的可控输入、可观察输出和噪声边界。
- 递推题的初值、模数、转移矩阵或可从 OEIS 验证的前几项。
- CAPTCHA/QR/polyglot 的结构约束、样本数量和自动化接口。
- 如果涉及 Web/crypto，确认主要障碍是否应转专项页面。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| comparison-only oracle | 反馈是否支持二分或逐字符搜索 | 写自动化查询，控制 rate/noise |
| timeout/error oracle | 时间或错误是否稳定区分条件 | Web 题转 SQLi/oracle 页 |
| recurrence/sequence | OEIS、特征多项式、矩阵幂和模数 | 建快速幂或闭式恢复 |
| finite-field recurrence samples | 转移多项式次数、样本数量和模数是否足够消元 | Groebner/Resultant 后再用 SMT 或正向复算 |
| finite-field affine protocol | `RESET` 后输出对输入呈仿射关系，且状态机反馈可反复采样 | 采零点/基向量恢复矩阵，求触发线，再固定 oracle 爆破小空间 |
| QR structural constraints | finder/timing/format 信息是否足够 | 枚举缺失块并校验 QR |
| CAPTCHA/font | 字体是否动态、图片是否可由 DOM/Canvas 获取 | Selenium/Tesseract/模板匹配 |
| modular CAPTCHA / expression oracle | CAPTCHA 是否可转 CRT/LLL 约束，表达式输出是否形成 crypto oracle | 先构造最短表达式，再转 crypto family |
| esolang/polyglot | 每层输出是否可作为下一层输入 | 分层保存中间产物 |
| hash length extension | MAC 形态是否为 `hash(secret || msg)` | 转 [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [ACTF2026-ohmycaptcha-wp](../raw/crypto/ACTF2026-ohmycaptcha-wp.md) | CAPTCHA 由模素数余数集合决定，可用 CRT+LLL 直接构造能通过校验的 Python 表达式，再把后续输出交给 crypto oracle。 |
| [HGAME2026-flux-wp](../raw/crypto/HGAME2026-flux-wp.md) | 4 个连续状态来自有限域二次递推；先 Groebner 消元恢复递推参数，再用 bit-vector SMT 反求自定义 hash key。 |
| [NCTF2026-ezprotocol-wp](../raw/misc/NCTF2026-ezprotocol-wp.md) | 自定义协议的状态机比加密更关键：恢复 XOR payload 和 checksum 后，利用同连接认证状态拼接多 packet。 |
| [SU_MirrorBus9WP](../raw/misc/SU_MirrorBus9WP.md) | 半双工协议可在 reset 态采样仿射映射；先解出能触发 CHAL 的 MIX 点，再利用固定 CHAL 对 16-bit `PROVE` checksum 分批穷举。 |
| [LilacCTF2026-nestdlp-wp](../raw/crypto/LilacCTF2026-nestdlp-wp.md) | 二元多项式商环 DLP 可转成乘法矩阵行列式上的 p-adic DLP；恢复指数后还要用 padding 汉明重量约束解明文。 |

## 合并与拆分结论

- 保留为 family：这些题共享“把反馈转为状态/约束”的首轮判断。
- 不合并进 `vm-z3-sandbox-and-game-basics.md`：该页是规则系统总入口，本页聚焦 oracle/递推/自动化识别。
- 不拆 CAPTCHA、递推、oracle 小页：当前 raw 更适合作为 misc 二级路由。

## 常见误判

- 不测 oracle 噪声和 rate limit，自动化脚本结果不稳定。
- 递推题只暴力前几项，没有找矩阵/闭式结构。
- CAPTCHA 只上 OCR，不利用字体、DOM 或生成规律。
- Polyglot 没保存中间产物，后续无法复盘哪层出错。

## 关联页面

- [misc-cross-category-triage-family.md](misc-cross-category-triage-family.md)
- [vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md)
- [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md)
- [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md)
- [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md)

## 原始资料

- [oracles-recurrences-captcha-polyglots.md](../raw/misc/oracles-recurrences-captcha-polyglots.md)
- [ACTF2026-ohmycaptcha-wp](../raw/crypto/ACTF2026-ohmycaptcha-wp.md)
- [HGAME2026-flux-wp](../raw/crypto/HGAME2026-flux-wp.md)
- [NCTF2026-ezprotocol-wp](../raw/misc/NCTF2026-ezprotocol-wp.md)
- [SU_MirrorBus9WP.md](../raw/misc/SU_MirrorBus9WP.md)
