---
type: family
tags: [misc, family, triage]
skills: [ctf-misc]
updated: 2026-06-11
---

# Misc Cross-Category Triage

## 适用场景

题目暂时不能归入单一专项，表现为编码链、游戏状态、受限 shell、交互 oracle、轻量附件、音频/物理信号或 sandbox 混合。

本页是 WP 摄入后的 pivot 页：它不按比赛或题目组织，而是把 raw WP 中重复出现的识别信号压缩成下一跳判断表。

## 识别信号

- 附件或服务很小，但存在明显交互规则、编码层、资源包、shell 限制或状态转移。
- 初始方向可能在 misc、forensics、web、pwn、reverse 之间摆动。
- 需要先判断载体、交互模型和约束，再进入具体技巧页。

## 最小证据

- 至少有一组可复算的参数、可重放的输入输出、可观察的错误/状态差异，或可定位的文件/交互边界。
- 能说明当前题更接近下表哪个下一跳技巧页，而不是只按比赛名或题名判断。
- 若多个变体同时出现，先选择能最小验证的信号；失败后再按相邻技巧 pivot。

## 解法骨架

1. 先确认载体：文件、文本、协议、shell、游戏、音频还是 sandbox。
2. 找最小可控输入和最小可观察输出。
3. 按表进入编码、游戏状态、restricted shell、pyjail、oracle/递推或文件首检页面。
4. 如果证据明显转向 web/pwn/reverse/forensics，及时 pivot 到专项 skill。

## 关键变体

| 下一跳 | 触发信号 | WP 数 |
|---|---|---:|
| [bgp-rpki-route-hijack.md](bgp-rpki-route-hijack.md) | BGP/RPKI/AS/CIDR 路由策略或 origin-only 校验。 | 1 |
| [malware-c2-session-key-and-protocol-recovery.md](malware-c2-session-key-and-protocol-recovery.md) | C2 agent、通信流量、内存密钥或协议解密。 | 1 |
| [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md) | VM、磁盘、内存镜像、容器或取证环境修复。 | 3 |
| [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md) | ZIP/磁盘/容器恢复，尤其是已知明文或结构字段辅助恢复。 | 1 |
| [dns.md](dns.md) | DNS/SPF/TXT 记录和域名解析链。 | 1 |
| [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) | 编码链、二维码、助记词、Unicode 或文本变换。 | 5 |
| [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md) | Unicode 归一化、异常格式或结构化数据清洗。 | 1 |
| [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md) | 轻量附件、压缩包、签到或一行式首检。 | 5 |
| [font-shader-firmware-and-legacy-patterns.md](font-shader-firmware-and-legacy-patterns.md) | 字体、shader、旧格式或资源包承载隐藏信息。 | 1 |
| [game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md) | 游戏状态、交互协议、资源包或客户端状态复现。 | 5 |
| [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md) | 图片通道、位平面、二维码、像素块或 JPEG/PNG 隐写。 | 3 |
| [interactive-containers-jails-and-solvers.md](interactive-containers-jails-and-solvers.md) | 容器、虚拟化边界、交互环境或运行时 helper 可被借用。 | 1 |
| [llm-attacks.md](llm-attacks.md) | LLM prompt injection、角色泄露或模型约束绕过。 | 1 |
| [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md) | 矩阵、递推、交互 oracle、约束求解或协议状态。 | 2 |
| [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) | 任意文件读、路径推断、上传/恢复或资源路径穿越。 | 1 |
| [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md) | PCAP、协议重组、量子/媒体数据流或网络证据恢复。 | 2 |
| [pdf-png-gif-and-text-stego.md](pdf-png-gif-and-text-stego.md) | GIF/PNG/PDF/文本多媒体隐写与拆帧。 | 1 |
| [pyjails.md](pyjails.md) | Python 受限执行、agent sandbox 或对象链逃逸。 | 4 |
| [race-condition-and-concurrency-exploits.md](race-condition-and-concurrency-exploits.md) | TOCTOU、并发窗口或状态同步延迟导致检查与使用不一致。 | 1 |
| [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) | 非 crypto 载体中暴露 RSA 参数、密文或弱模数，需要跨方向 pivot。 | 1 |
| [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md) | 公开主页、平台账号、历史记录、媒体来源或身份链路 OSINT。 | 3 |
| [source-backdoors-and-restricted-shell-tricks.md](source-backdoors-and-restricted-shell-tricks.md) | SSH/受限 shell/源码后门/命令副作用。 | 4 |
| [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) | Web 前端参数、接口状态或业务流直接决定结果。 | 1 |
| [blockchain-smart-contract-exploitation.md](blockchain-smart-contract-exploitation.md) | 智能合约、链上伪随机、地址生成、账户/capability 绑定或交易调用约束。 | 5 |
| [windows-registry-logs-and-credentials.md](windows-registry-logs-and-credentials.md) | Windows/Chrome/DPAPI 凭据、注册表或日志提取。 | 1 |

## WP 案例沉淀

本节只抽取 raw WP 中可复用的识别信号和下一跳，不替代原始题解正文。

| Raw WP | 复用信号 | 下一跳 |
|---|---|---|
| [ACTF2026-agent-wp](../raw/misc/ACTF2026-agent-wp.md) | Python/agent/sandbox 行为可控，先枚举可用对象、过滤规则和逃逸原语。 | [pyjails.md](pyjails.md) |
| [ACTF2026-ezssh-wp](../raw/misc/ACTF2026-ezssh-wp.md) | 受限 shell、SSH 或源码后门，先找允许命令副作用、配置泄露和隐藏触发条件。 | [source-backdoors-and-restricted-shell-tricks.md](source-backdoors-and-restricted-shell-tricks.md) |
| [ACTF2026-farthest2026-wp](../raw/misc/ACTF2026-farthest2026-wp.md) | DOS/dosemu2/COMCOM64 sidecar 形成容器边界逃逸，先确认虚拟化边界、helper 加载和 host 侧执行点。 | [interactive-containers-jails-and-solvers.md](interactive-containers-jails-and-solvers.md) |
| [ACTF2026-master-of-album-wp](../raw/misc/ACTF2026-master-of-album-wp.md) | 游戏/交互状态或资源包可控，先把状态变化转成可重放脚本。 | [game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md) |
| [ACTF2026-questionnaire-wp](../raw/misc/ACTF2026-questionnaire-wp.md) | 问卷反馈型签到没有技术利用链，先确认交互流程和返回页面，不要误挖不存在的附件。 | [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md) |
| [ACTF2026-special-day-wp](../raw/misc/ACTF2026-special-day-wp.md) | 编码链、助记词、签到或日期/文本变换，先识别层级并逐层验证可逆性。 | [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) |
| [ACTF2026-zjuam-just-uses-awful-math-wp](../raw/misc/ACTF2026-zjuam-just-uses-awful-math-wp.md) | HTTP 抓包里暴露弱 RSA 公钥和密文，虽然归在 misc raw，首轮应 pivot 到 crypto/RSA 参数分解。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [HGAME2026-打好基础-wp](../raw/misc/HGAME2026-打好基础-wp.md) | 编码链、助记词、签到或日期/文本变换，先识别层级并逐层验证可逆性。 | [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) |
| [HGAME2026-invest-on-matrix-wp](../raw/misc/HGAME2026-invest-on-matrix-wp.md) | 矩阵、递推、协议或约束求解，先抽象状态转移和最小查询。 | [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md) |
| [HGAME2026-redacted-wp](../raw/misc/HGAME2026-redacted-wp.md) | 轻量附件、压缩包或事件材料首检，先查 magic、metadata、嵌套和一行式验证路径。 | [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md) |
| [HGAME2026-shiori不想找女友-wp](../raw/misc/HGAME2026-shiori不想找女友-wp.md) | Python/agent/sandbox 行为可控，先枚举可用对象、过滤规则和逃逸原语。 | [pyjails.md](pyjails.md) |
| [HGAME2026-vidar-token-wp](../raw/misc/HGAME2026-vidar-token-wp.md) | 轻量附件、压缩包或事件材料首检，先查 magic、metadata、嵌套和一行式验证路径。 | [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md) |
| [LilacCTF2026-incident-wp](../raw/misc/LilacCTF2026-incident-wp.md) | Solana runtime direct mapping 与 CPI 状态同步边界可被利用，先确认账户 owner、writable 位和异常账户构造。 | [blockchain-smart-contract-exploitation.md](blockchain-smart-contract-exploitation.md) |
| [LilacCTF2026-launchpad-wp](../raw/misc/LilacCTF2026-launchpad-wp.md) | Solana/Anchor PDA、mint、receipt 和 claim request 约束未绑定同一状态组，先审计账户关系而非游戏状态。 | [blockchain-smart-contract-exploitation.md](blockchain-smart-contract-exploitation.md) |
| [NCTF2026-ezprotocol-wp](../raw/misc/NCTF2026-ezprotocol-wp.md) | 矩阵、递推、协议或约束求解，先抽象状态转移和最小查询。 | [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md) |
| [NCTF2026-merlin-wp](../raw/misc/NCTF2026-merlin-wp.md) | Merlin Agent dump 与通信流量组合，先从内存中提取 OPAQUE 会话密钥再解密 C2 payload。 | [malware-c2-session-key-and-protocol-recovery.md](malware-c2-session-key-and-protocol-recovery.md) |
| [NCTF2026-quantum-vault-wp](../raw/misc/NCTF2026-quantum-vault-wp.md) | 业务同步延迟和 SUID 程序 lstat/open 窗口都属于 TOCTOU，先固定检查与使用之间可替换的对象。 | [race-condition-and-concurrency-exploits.md](race-condition-and-concurrency-exploits.md) |
| [NCTF2026-what-a-mess-wp](../raw/misc/NCTF2026-what-a-mess-wp.md) | 表格字段被 Unicode、全角、零宽字符和格式变体污染，先做规范化、白名单和校验位统计。 | [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md) |
| [RCTF2025-514-wp](../raw/misc/RCTF2025-514-wp.md) | 编码链、助记词、签到或日期/文本变换，先识别层级并逐层验证可逆性。 | [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) |
| [RCTF2025-asgard-fallen-down-wp](../raw/misc/RCTF2025-asgard-fallen-down-wp.md) | 受限 shell、SSH 或源码后门，先找允许命令副作用、配置泄露和隐藏触发条件。 | [source-backdoors-and-restricted-shell-tricks.md](source-backdoors-and-restricted-shell-tricks.md) |
| [RCTF2025-shadows-of-asgard-wp](../raw/misc/RCTF2025-shadows-of-asgard-wp.md) | 受限 shell、SSH 或源码后门，先找允许命令副作用、配置泄露和隐藏触发条件。 | [source-backdoors-and-restricted-shell-tricks.md](source-backdoors-and-restricted-shell-tricks.md) |
| [RCTF2025-signin-wp](../raw/misc/RCTF2025-signin-wp.md) | 前端源码把完成度作为 `score` 参数提交，先读业务逻辑并直接验证阈值触发条件。 | [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) |
| [RCTF2025-speak-softly-love-wp](../raw/misc/RCTF2025-speak-softly-love-wp.md) | 音乐视频、个人主页、音频文件、SVN 记录和 gopher 页面串联，先保存公开媒体和站点证据链。 | [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md) |
| [RCTF2025-the-alchemists-cage-wp](../raw/misc/RCTF2025-the-alchemists-cage-wp.md) | LLM 隐藏提示词泄露题，先确认角色/权限叙事能否诱导模型复述系统提示或历史对话。 | [llm-attacks.md](llm-attacks.md) |
| [RCTF2025-vault-wp](../raw/misc/RCTF2025-vault-wp.md) | Sui Move 共享 `TreasuryCap` 导致任意 mint，先审计资源所有权、capability 和购买 flag 条件。 | [blockchain-smart-contract-exploitation.md](blockchain-smart-contract-exploitation.md) |
| [RCTF2025-wanna-feel-love-wp](../raw/misc/RCTF2025-wanna-feel-love-wp.md) | spammimic、XM sample、Ghostarchive、PayPal 和墓碑页面串成多阶段 OSINT，先记录每个公开来源。 | [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md) |
| [SU_Artifact_OnlineWP](../raw/misc/SU_Artifact_OnlineWP.md) | 轻量附件、压缩包或事件材料首检，先查 magic、metadata、嵌套和一行式验证路径。 | [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md) |
| [SU_chaosWP](../raw/misc/SU_chaosWP.md) | ZipCrypto 已知明文攻击依赖 AVIF/ISOBMFF 头部结构，先确认稳定明文字段再跑恢复工具。 | [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md) |
| [SU_MirrorBus9WP](../raw/misc/SU_MirrorBus9WP.md) | 游戏/交互状态或资源包可控，先把状态变化转成可重放脚本。 | [game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md) |
| [SU_SigninWP](../raw/misc/SU_SigninWP.md) | 题面给出 CTFtime 团队页面，先访问公开资料并处理页面中的 flag-like 字符串。 | [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md) |
| [VNCTF2026-ez-iot-wp](../raw/misc/VNCTF2026-ez-iot-wp.md) | 轻量附件、压缩包或事件材料首检，先查 magic、metadata、嵌套和一行式验证路径。 | [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md) |
| [VNCTF2026-huntingagent-wp](../raw/misc/VNCTF2026-huntingagent-wp.md) | Python/agent/sandbox 行为可控，先枚举可用对象、过滤规则和逃逸原语。 | [pyjails.md](pyjails.md) |
| [VNCTF2026-minecraft-wp](../raw/misc/VNCTF2026-minecraft-wp.md) | 游戏/交互状态或资源包可控，先把状态变化转成可重放脚本。 | [game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md) |
| [VNCTF2026-mymnemonic-wp](../raw/misc/VNCTF2026-mymnemonic-wp.md) | 编码链、助记词、签到或日期/文本变换，先识别层级并逐层验证可逆性。 | [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) |
| [VNCTF2026-vnshell-wp](../raw/misc/VNCTF2026-vnshell-wp.md) | 受限 shell、SSH 或源码后门，先找允许命令副作用、配置泄露和隐藏触发条件。 | [source-backdoors-and-restricted-shell-tricks.md](source-backdoors-and-restricted-shell-tricks.md) |
| [D3CTF2019-bet2loss-v2-wp](../raw/misc/D3CTF2019-bet2loss-v2-wp.md) | 以太坊合约泄露 croupier 私钥且 settle 可重放，先确认签名能力、伪随机和 constructor 绕过。 | [blockchain-smart-contract-exploitation.md](blockchain-smart-contract-exploitation.md) |
| [D3CTF2019-c-c-wp](../raw/misc/D3CTF2019-c-c-wp.md) | 私用区字符依赖动态加载字体，先脱壳跟进字体 hook 并 dump 解密后的字体资源。 | [font-shader-firmware-and-legacy-patterns.md](font-shader-firmware-and-legacy-patterns.md) |
| [D3CTF2019-find-me-wp](../raw/misc/D3CTF2019-find-me-wp.md) | Chrome Login Data 和 lsass.dmp 组合，先修复嵌入 ZIP 再提取 DPAPI master key 解密浏览器凭据。 | [windows-registry-logs-and-credentials.md](windows-registry-logs-and-credentials.md) |
| [D3CTF2019-vera-wp](../raw/misc/D3CTF2019-vera-wp.md) | VeraCrypt 容器和图像光栅恢复组合，先用题面线索找密码再处理解出的媒体 artifact。 | [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md) |
| [D3CTF2021-easy-quantum-wp](../raw/misc/D3CTF2021-easy-quantum-wp.md) | PCAP 中传输 pickle/numpy 量子态和测量基，先按流量结构恢复 BB84 密钥再异或密文。 | [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md) |
| [D3CTF2021-robust-wp](../raw/misc/D3CTF2021-robust-wp.md) | HTTP/3/QUIC 流量和媒体信号混合，先重组网络层数据，再转频谱或 payload 提取。 | [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md) |
| [D3CTF2021-scientific-calculator-wp](../raw/misc/D3CTF2021-scientific-calculator-wp.md) | CPython audit hook 沙箱绑定精确版本，先记录允许 action、compile/exec 边界和公开资料缺口。 | [pyjails.md](pyjails.md) |
| [D3CTF2021-shellgen2-wp](../raw/misc/D3CTF2021-shellgen2-wp.md) | 无字母数字 PHP 生成器本质是受限字符表达式构造，先建字符索引表和递增优化。 | [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) |
| [D3CTF2021-virtual-love-revenge-2-0-wp](../raw/misc/D3CTF2021-virtual-love-revenge-2-0-wp.md) | VMware 配置损坏、零宽字符字典和单用户模式取证组合，先修复虚拟机再进入磁盘证据链。 | [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md) |
| [D3CTF2022-badw3ter-wp](../raw/misc/D3CTF2022-badw3ter-wp.md) | 图片图层、黑底 QR 和隐写链组合，先分离图层/背景再恢复二维码或图像隐藏信息。 | [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md) |
| [D3CTF2022-ohhhh-spf-wp](../raw/misc/D3CTF2022-ohhhh-spf-wp.md) | SPF/DNS 记录是主线，先查询 TXT/SPF 解析链并确认域名授权语义。 | [dns.md](dns.md) |
| [D3CTF2022-wannawacca-wp](../raw/misc/D3CTF2022-wannawacca-wp.md) | Windows 内存镜像、勒索样本和流量协议复现组合，先 dump 可疑进程和文件再还原加密链。 | [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md) |
| [D3CTF2023-d3casino-wp](../raw/misc/D3CTF2023-d3casino-wp.md) | 链上伪随机可预测但限制合约代码大小，先用 vanity 地址、minimal proxy 和 CREATE2 控制调用者。 | [blockchain-smart-contract-exploitation.md](blockchain-smart-contract-exploitation.md) |
| [D3CTF2023-d3craft-wp](../raw/misc/D3CTF2023-d3craft-wp.md) | Minecraft/PaperMC 移动事件和数据包语义差异可绕过，先逆插件目标位置再脚本化状态移动。 | [game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md) |
| [D3CTF2023-d3gif-wp](../raw/misc/D3CTF2023-d3gif-wp.md) | GIF/PNG/QR 媒体层叠，先拆帧和图像通道，再恢复二维码或文本 artifact。 | [pdf-png-gif-and-text-stego.md](pdf-png-gif-and-text-stego.md) |
| [D3CTF2023-d3image-wp](../raw/misc/D3CTF2023-d3image-wp.md) | 图像/二维码/摩斯或像素模式是主线，先分离颜色层、定位图案再恢复可读编码。 | [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md) |
| [D3CTF2023-d3readfile-wp](../raw/misc/D3CTF2023-d3readfile-wp.md) | 任意文件读不知道 flag 路径时，先读取 locate 数据库并在本地搜索目标路径。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| [D3CTF2025-d3image-wp](../raw/misc/D3CTF2025-d3image-wp.md) | 图像块变换和隐写编码可逆，先抽出块顺序/颜色关系再写正反向恢复脚本。 | [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md) |
| [D3CTF2025-d3rpg-signin-wp](../raw/misc/D3CTF2025-d3rpg-signin-wp.md) | RPG 地图、标牌、隐藏路径和摩斯地板是主线，先复现游戏状态和资源线索。 | [game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md) |
| [D3CTF2025-d3rpki-wp](../raw/misc/D3CTF2025-d3rpki-wp.md) | RPKI/BGP 只校验 origin 时可伪造 peer 或宣告更具体路由，先确认 AS、CIDR 和 filter。 | [bgp-rpki-route-hijack.md](bgp-rpki-route-hijack.md) |

## 常见陷阱

- 把 WP 当完整题解目录使用，没有提取可迁移的触发信号。
- 只按题名命中下一跳，忽略源码、参数规模、可控输入和错误反馈。
- 已有技巧页能覆盖时仍新建单题页面，导致 wiki 退化成 writeup 列表。

## 关联技巧

- [bgp-rpki-route-hijack.md](bgp-rpki-route-hijack.md)
- [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)
- [dns.md](dns.md)
- [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md)
- [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md)
- [font-shader-firmware-and-legacy-patterns.md](font-shader-firmware-and-legacy-patterns.md)
- [game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md)
- [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md)
- [interactive-containers-jails-and-solvers.md](interactive-containers-jails-and-solvers.md)
- [keyboard-mouse-audio-and-physical-puzzles.md](keyboard-mouse-audio-and-physical-puzzles.md)
- [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md)
- [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)
- [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md)
- [pdf-png-gif-and-text-stego.md](pdf-png-gif-and-text-stego.md)
- [pyjails.md](pyjails.md)
- [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)
- [source-backdoors-and-restricted-shell-tricks.md](source-backdoors-and-restricted-shell-tricks.md)
- [blockchain-smart-contract-exploitation.md](blockchain-smart-contract-exploitation.md)
- [windows-registry-logs-and-credentials.md](windows-registry-logs-and-credentials.md)

## 原始资料

本页原始资料以“WP 案例沉淀”表中的 raw WP 链接为准，共 55 篇。
