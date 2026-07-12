---
type: family
tags: [cross-category, family, triage]
skills: [ctf-solve-challenge]
updated: 2026-07-11
---

# Cross-Category Triage

## 作用边界

题目暂时不能归入单一专项，表现为游戏状态、受限 shell、交互 oracle、轻量附件、音频/物理信号、sandbox 或多个正式方向混合。这里是 `ctf-solve-challenge` 的继续分流状态，不是正式方向，也不是万能执行层；普通表示层编码本身直接转 Crypto。

本页是 WP 摄入后的 pivot 页：它不按比赛或题目组织，而是把 raw WP 中重复出现的识别信号压缩成下一跳判断表。

## 识别信号

- 附件或服务很小，但存在明显交互规则、编码层、资源包、shell 限制或状态转移。
- 初始方向可能在 cross-category、forensics、web、pwn、reverse 或 crypto 之间摆动。
- 需要先判断载体、交互模型和约束，再进入具体技巧页。

## 最小证据

- 至少有一组可复算的参数、可重放的输入输出、可观察的错误/状态差异，或可定位的文件/交互边界。
- 能说明当前题更接近下表哪个下一跳技巧页，而不是只按比赛名或题名判断。
- 若多个变体同时出现，先选择能最小验证的信号；失败后再按相邻技巧 pivot。

## 分流流程

1. 先确认载体：文件、文本、协议、shell、游戏、音频还是 sandbox。
2. 找最小可控输入和最小可观察输出。
3. 按表进入 Crypto 编码、游戏状态、restricted shell、pyjail、oracle/递推或文件首检页面。
4. 如果证据明显转向 web/pwn/reverse/forensics，及时 pivot 到专项 skill。

## Cross-category 路线分流

同一 raw WP 可能同时支撑多个下一跳，因此“案例触达”统计的是路由关系，不是唯一 WP 篇数。

| 下一跳 | 触发信号 | 案例触达 |
|---|---|---:|
| [bgp-rpki-route-hijack.md](bgp-rpki-route-hijack.md) | BGP/RPKI/AS/CIDR 路由策略或 origin-only 校验。 | 1 |
| [malware-c2-session-key-and-protocol-recovery.md](malware-c2-session-key-and-protocol-recovery.md) | C2 agent、通信流量、内存密钥或协议解密。 | 4 |
| [linux-git-browser-and-container-forensics.md](linux-git-browser-and-container-forensics.md) | Linux 日志、Git 对象、浏览器/容器历史或残留凭据。 | 2 |
| [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md) | VM、磁盘、内存镜像、容器或取证环境修复。 | 3 |
| [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md) | ZIP/磁盘/容器恢复，尤其是已知明文或结构字段辅助恢复。 | 3 |
| [dns.md](dns.md) | DNS/SPF/TXT 记录和域名解析链。 | 1 |
| [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) | 编码链、二维码、Unicode、符文映射或文本变换。 | 5 |
| [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md) | Unicode 归一化、异常格式或结构化数据清洗。 | 1 |
| [exotic-secret-sharing-rabin-and-polynomials.md](exotic-secret-sharing-rabin-and-polynomials.md) | BIP39、secret sharing、Rabin 或长尾代数结构恢复。 | 1 |
| [font-shader-firmware-and-legacy-patterns.md](font-shader-firmware-and-legacy-patterns.md) | 字体、shader、旧格式或资源包承载隐藏信息。 | 1 |
| [game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md) | 游戏状态、交互协议、资源包或客户端状态复现。 | 5 |
| [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md) | 低频架构、固件、MCU、无线设备或硬件执行语义。 | 1 |
| [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md) | 图片通道、位平面、二维码、像素块或 JPEG/PNG 隐写。 | 6 |
| [interactive-containers-jails-and-solvers.md](interactive-containers-jails-and-solvers.md) | 容器、虚拟化边界、交互环境或运行时 helper 可被借用。 | 1 |
| [llm-attacks.md](llm-attacks.md) | LLM prompt injection、角色泄露、工具调用滥用或模型约束绕过。 | 3 |
| [node-and-prototype.md](node-and-prototype.md) | Node 原型污染、服务端 JS gadget 或 VM sandbox escape。 | 2 |
| [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md) | 矩阵、递推、交互 oracle、约束求解或协议状态。 | 2 |
| [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) | 任意文件读、路径推断、上传/恢复或资源路径穿越。 | 1 |
| [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md) | PCAP、协议重组、量子/媒体数据流或网络证据恢复。 | 8 |
| [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md) | 已恢复凭据后继续 SSH/SFTP/内网服务横向或跨服务权限链。 | 2 |
| [pdf-png-gif-and-text-stego.md](pdf-png-gif-and-text-stego.md) | GIF/PNG/PDF/文本多媒体隐写与拆帧。 | 2 |
| [pyjails.md](pyjails.md) | Python 受限执行、agent sandbox 或对象链逃逸。 | 1 |
| [scripts-and-obfuscation.md](scripts-and-obfuscation.md) | 邮件附件、SVG/JS、脚本混淆、诱饵 flag 或 staged payload。 | 1 |
| [race-condition-and-concurrency-exploits.md](race-condition-and-concurrency-exploits.md) | TOCTOU、并发窗口或状态同步延迟导致检查与使用不一致。 | 1 |
| [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) | 非 crypto 载体中暴露 RSA 参数、密文或弱模数，需要跨方向 pivot。 | 1 |
| [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md) | 公开主页、平台账号、历史记录、媒体来源或身份链路 OSINT。 | 3 |
| [source-backdoors-and-restricted-shell-tricks.md](source-backdoors-and-restricted-shell-tricks.md) | SSH/受限 shell/源码后门/命令副作用。 | 1 |
| [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) | Web 前端参数、接口状态或业务流直接决定结果。 | 3 |
| [known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md) | 组件版本、补丁边界、公开漏洞或 N-day PoC 是主入口。 | 1 |
| [blockchain-smart-contract-exploitation.md](blockchain-smart-contract-exploitation.md) | 智能合约、链上伪随机、地址生成、账户/capability 绑定或交易调用约束。 | 6 |
| [windows-registry-logs-and-credentials.md](windows-registry-logs-and-credentials.md) | Windows/Chrome/DPAPI 凭据、注册表或日志提取。 | 1 |
| [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md) | 浏览器渲染、DOM sink、admin bot、截图或外带上下文。 | 1 |

## WP 案例沉淀

本节只抽取 raw WP 中可复用的识别信号和下一跳，不替代原始题解正文。

| Raw WP | 复用信号 | 下一跳 |
|---|---|---|
| [WMCTF2025-githacker-wp](../raw/forensics/WMCTF2025-githacker-wp.md) | `git log` 不完整但 `.git` 仍在，先看 reflog/dangling object；疑似图片高熵且大小规整时按 VeraCrypt 容器和卷头恢复处理。 | [linux-git-browser-and-container-forensics.md](linux-git-browser-and-container-forensics.md)、[filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md) |
| [WMCTF2025-shopping-company-phishing-email-wp](../raw/pentest/WMCTF2025-shopping-company-phishing-email-wp.md) | 客服 AI 工具调用可被 prompt injection 诱导解压/执行附件；`.eml` 中 SVG 附件应当作可执行 XML/JS 文档分析，注意诱饵 flag。 | [llm-attacks.md](llm-attacks.md)、[scripts-and-obfuscation.md](scripts-and-obfuscation.md) |
| [WMCTF2025-voice-hacker-wp](../raw/forensics/WMCTF2025-voice-hacker-wp.md) | 语音认证页面和后端接口不一致，PCAP 中 UDP 可 Decode As RTP 导出参考音频，再按后端 wav 特征伪造提交。 | [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md)、[audio-frequency-and-archive-stego.md](audio-frequency-and-archive-stego.md) |
| [ACTF2026-agent-wp](../raw/web/ACTF2026-agent-wp.md) | Node 服务把用户 `field` 拼进 dotted YAML path；污染 `constructor.prototype` 后改策略 profile，让 compat 公式解释器走到 `eval` 读 `/flag`。 | [node-and-prototype.md](node-and-prototype.md)、[web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) |
| [ACTF2026-ezssh-wp](../raw/pentest/ACTF2026-ezssh-wp.md) | SSH ForceCommand/team gate 只是入口；关键链路是 Debian OpenSSL 弱 key、`.bash_history`、Git dangling `.env` 和 SFTP-only 备份 unit 串联到内部 API。 | [linux-git-browser-and-container-forensics.md](linux-git-browser-and-container-forensics.md)、[source-backdoors-and-restricted-shell-tricks.md](source-backdoors-and-restricted-shell-tricks.md)、[pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md) |
| [ACTF2026-farthest2026-wp](../raw/pwn/ACTF2026-farthest2026-wp.md) | DOS/dosemu2/COMCOM64 sidecar 形成容器边界逃逸，先确认虚拟化边界、helper 加载和 host 侧执行点。 | [interactive-containers-jails-and-solvers.md](interactive-containers-jails-and-solvers.md) |
| [ACTF2026-master-of-album-wp](../raw/web/ACTF2026-master-of-album-wp.md) | Socket.IO 限时专辑问答可直接构造随机 `team/token/session_id` 开局；核心是封面/音频识别缓存和答题自动化。 | [game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md) |
| [ACTF2026-special-day-wp](../raw/crypto/ACTF2026-special-day-wp.md) | 极小文本附件是标准 Base64，解码后按题面规则去标点、用下划线连接节日短句构造 flag。 | [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) |
| [ACTF2026-zjuam-just-uses-awful-math-wp](../raw/crypto/ACTF2026-zjuam-just-uses-awful-math-wp.md) | HTTP 抓包里暴露弱 RSA 公钥和密文；载体是 HTTP，但决定性主障碍是 Crypto/RSA 参数分解。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [HGAME2026-打好基础-wp](../raw/crypto/HGAME2026-打好基础-wp.md) | Emoji 高密度文本指向 base100；每层输出继续落在 Base 系列字符集，按 base100 -> base92 -> ... -> base32 剥离。 | [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) |
| [HGAME2026-invest-on-matrix-wp](../raw/stego/HGAME2026-invest-on-matrix-wp.md) | 25 组 hint 是 `5x5` 二值 QR 小块按行扁平化；按 `pts-1` 的块坐标拼回 `25x25` 二维码，再扫码进入最终验证。 | [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md) |
| [HGAME2026-redacted-wp](../raw/forensics/HGAME2026-redacted-wp.md) | PDF 黑条遮挡不等于文本删除；复制文本、ToUnicode/CMap 字体映射、Inkscape 隐藏层和增量保存 `%%EOF` 都是首轮证据。 | [pdf-png-gif-and-text-stego.md](pdf-png-gif-and-text-stego.md) |
| [HGAME2026-shiori不想找女友-wp](../raw/stego/HGAME2026-shiori不想找女友-wp.md) | PNG EXIF 中给出 `start/step/column` JSON 参数；按规则噪点网格抽样重排像素，恢复隐藏图像线索。 | [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md) |
| [HGAME2026-vidar-token-wp](../raw/blockchain/HGAME2026-vidar-token-wp.md) | 前端暴露 RPC 与 WASM 解密逻辑，链上 `tokenURI` 指向魔改 ERC-20；`decimals` 和 `symbol` 分别承载参数和密文。 | [blockchain-smart-contract-exploitation.md](blockchain-smart-contract-exploitation.md) |
| [LilacCTF2026-incident-wp](../raw/blockchain/LilacCTF2026-incident-wp.md) | Solana runtime direct mapping 与 CPI 状态同步边界可被利用，先确认账户 owner、writable 位和异常账户构造。 | [blockchain-smart-contract-exploitation.md](blockchain-smart-contract-exploitation.md) |
| [LilacCTF2026-launchpad-wp](../raw/blockchain/LilacCTF2026-launchpad-wp.md) | Solana/Anchor PDA、mint、receipt 和 claim request 约束未绑定同一状态组，先审计账户关系而非游戏状态。 | [blockchain-smart-contract-exploitation.md](blockchain-smart-contract-exploitation.md) |
| [NCTF2026-ezprotocol-wp](../raw/pwn/NCTF2026-ezprotocol-wp.md) | 泄露协议定义给出 10 字节 header、CRC32 和 XOR key `NCTF`；同连接认证状态可复用，拼接 Auth 与 GetFlag packet 即可绕过密码检查。 | [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md)、[pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md) |
| [NCTF2026-merlin-wp](../raw/malware/NCTF2026-merlin-wp.md) | Merlin Agent dump 与通信流量组合，先从内存中提取 OPAQUE 会话密钥再解密 C2 payload。 | [malware-c2-session-key-and-protocol-recovery.md](malware-c2-session-key-and-protocol-recovery.md) |
| [NCTF2026-quantum-vault-wp](../raw/pwn/NCTF2026-quantum-vault-wp.md) | 业务同步延迟和 SUID 程序 lstat/open 窗口都属于 TOCTOU，先固定检查与使用之间可替换的对象。 | [race-condition-and-concurrency-exploits.md](race-condition-and-concurrency-exploits.md) |
| [NCTF2026-what-a-mess-wp](../raw/stego/NCTF2026-what-a-mess-wp.md) | 本库将格式变体掩饰记录语义视为 Stego 边界案例；实操仍是 Unicode/全角/零宽字符规范化、白名单和校验位统计。 | [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md) |
| [RCTF2025-514-wp](../raw/web/RCTF2025-514-wp.md) | Koishi/Puppeteer 截图把未过滤 text 写入本地 `file://` 页面；注入 JS 创建 `iframe file:///flag` 并覆盖 canvas 回显。 | [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)、[xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md) |
| [RCTF2025-asgard-fallen-down-wp](../raw/malware/RCTF2025-asgard-fallen-down-wp.md) | PCAP 中 `build/version` 可恢复 AES key/iv；解密 `X-Cache-Data` 和 `/contact` 分片后串起命令回显、环境变量和 base64 图像证据。 | [malware-c2-session-key-and-protocol-recovery.md](malware-c2-session-key-and-protocol-recovery.md)、[pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md) |
| [RCTF2025-shadows-of-asgard-wp](../raw/malware/RCTF2025-shadows-of-asgard-wp.md) | PCAP 导出伪装落地页、加密 C2 JSON 和 PNG `tEXt` chunk；逐题把公司名、注入路径、task id、时间和 base64 flag 绑定到 artifact。 | [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md)、[malware-c2-session-key-and-protocol-recovery.md](malware-c2-session-key-and-protocol-recovery.md) |
| [RCTF2025-signin-wp](../raw/web/RCTF2025-signin-wp.md) | 前端源码把完成度作为 `score` 参数提交，先读业务逻辑并直接验证阈值触发条件。 | [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) |
| [RCTF2025-speak-softly-love-wp](../raw/osint/RCTF2025-speak-softly-love-wp.md) | 音乐视频、个人主页、音频文件、SVN 记录和 gopher 页面串联，先保存公开媒体和站点证据链。 | [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md) |
| [RCTF2025-the-alchemists-cage-wp](../raw/ai-ml/RCTF2025-the-alchemists-cage-wp.md) | LLM 隐藏提示词泄露题，先确认角色/权限叙事能否诱导模型复述系统提示或历史对话。 | [llm-attacks.md](llm-attacks.md) |
| [RCTF2025-vault-wp](../raw/blockchain/RCTF2025-vault-wp.md) | Sui Move 共享 `TreasuryCap` 导致任意 mint，先审计资源所有权、capability 和购买 flag 条件。 | [blockchain-smart-contract-exploitation.md](blockchain-smart-contract-exploitation.md) |
| [RCTF2025-wanna-feel-love-wp](../raw/osint/RCTF2025-wanna-feel-love-wp.md) | spammimic、XM sample、Ghostarchive、PayPal 和墓碑页面串成多阶段 OSINT，先记录每个公开来源。 | [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md) |
| [SUCTF2026-Artifact_OnlineWP](../raw/stego/SUCTF2026-Artifact_OnlineWP.md) | 本库将符文与 cube 空间状态中的命令隐藏视为 Stego 边界案例；核心求解仍是建模 `R/C/F` 置换后搜索可执行命令。 | [game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md)、[encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) |
| [SUCTF2026-chaosWP](../raw/crypto/SUCTF2026-chaosWP.md) | ZipCrypto 已知明文攻击依赖 AVIF/ISOBMFF 头部结构，先确认稳定明文字段再跑恢复工具。 | [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md) |
| [SUCTF2026-MirrorBus9WP](../raw/crypto/SUCTF2026-MirrorBus9WP.md) | Half-duplex bus 的 `RESET/ENQ/ARM/PROVE` 可建模为 `F_65521` 仿射协议；先采基点学习矩阵，再对 CHAL checksum 做固定目标爆破。 | [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md) |
| [SUCTF2026-SigninWP](../raw/osint/SUCTF2026-SigninWP.md) | 题面给出 CTFtime 团队页面，先访问公开资料并处理页面中的 flag-like 字符串。 | [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md) |
| [VNCTF2026-ez-iot-wp](../raw/hardware-embedded/VNCTF2026-ez-iot-wp.md) | Xtensa ESP 固件保留 `sender_task`，可恢复 ESP-NOW 应用层包头、分片字段、IV 与 AES-CBC 密文，再从 `capture.raw` 重组 PNG。 | [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)、[pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md) |
| [VNCTF2026-huntingagent-wp](../raw/ai-ml/VNCTF2026-huntingagent-wp.md) | Supervisor 只审前 600 字节，Coordinator 可被诱导调用 Skill；前半 flag 来自 prompt/Skill 泄露，后半靠 Node `vm.runInNewContext` 构造器链逃逸。 | [llm-attacks.md](llm-attacks.md)、[node-and-prototype.md](node-and-prototype.md) |
| [VNCTF2026-minecraft-wp](../raw/pentest/VNCTF2026-minecraft-wp.md) | Minecraft/Velocity 多子服共用 LuckPerms PostgreSQL；creative 命令方块矿车提 OP 后改代理权限，再切 survival 利用旧 Paper/JDK Log4Shell。 | [game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md)、[pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md)、[known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md) |
| [VNCTF2026-mymnemonic-wp](../raw/crypto/VNCTF2026-mymnemonic-wp.md) | 图片末尾黑白格按 10 像素步长提取 192-bit ENT；补 BIP39 checksum 后映射中文 18 词助记词并派生 seed。 | [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md)、[exotic-secret-sharing-rabin-and-polynomials.md](exotic-secret-sharing-rabin-and-polynomials.md) |
| [VNCTF2026-vnshell-wp](../raw/malware/VNCTF2026-vnshell-wp.md) | WebShell 流量落下 stage1，`gift` 以 XOR `0x99` 拉取 Go stage2；再解 AES-CBC 配置、AES-GCM C2 流量和 ZipCrypto `VIP_file`。 | [malware-c2-session-key-and-protocol-recovery.md](malware-c2-session-key-and-protocol-recovery.md)、[pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md)、[filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md) |
| [D3CTF2019-bet2loss-v2-wp](../raw/blockchain/D3CTF2019-bet2loss-v2-wp.md) | 以太坊合约泄露 croupier 私钥且 settle 可重放，先确认签名能力、伪随机和 constructor 绕过。 | [blockchain-smart-contract-exploitation.md](blockchain-smart-contract-exploitation.md) |
| [D3CTF2019-c-c-wp](../raw/reverse/D3CTF2019-c-c-wp.md) | 私用区字符依赖动态加载字体，先脱壳跟进字体 hook 并 dump 解密后的字体资源。 | [font-shader-firmware-and-legacy-patterns.md](font-shader-firmware-and-legacy-patterns.md) |
| [D3CTF2019-find-me-wp](../raw/forensics/D3CTF2019-find-me-wp.md) | Chrome Login Data 和 lsass.dmp 组合，先修复嵌入 ZIP 再提取 DPAPI master key 解密浏览器凭据。 | [windows-registry-logs-and-credentials.md](windows-registry-logs-and-credentials.md) |
| [D3CTF2019-vera-wp](../raw/forensics/D3CTF2019-vera-wp.md) | VeraCrypt 容器和图像光栅恢复组合，先用题面线索找密码再处理解出的媒体 artifact。 | [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md) |
| [D3CTF2021-easy-quantum-wp](../raw/crypto/D3CTF2021-easy-quantum-wp.md) | PCAP 中传输 pickle/numpy 量子态和测量基，先按流量结构恢复 BB84 密钥再异或密文。 | [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md) |
| [D3CTF2021-robust-wp](../raw/forensics/D3CTF2021-robust-wp.md) | HTTP/3/QUIC 流量和媒体信号混合，先重组网络层数据，再转频谱或 payload 提取。 | [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md) |
| [D3CTF2021-scientific-calculator-wp](../raw/pwn/D3CTF2021-scientific-calculator-wp.md) | CPython audit hook 沙箱绑定精确版本，先记录允许 action、compile/exec 边界和公开资料缺口。 | [pyjails.md](pyjails.md) |
| [D3CTF2021-shellgen2-wp](../raw/web/D3CTF2021-shellgen2-wp.md) | 无字母数字 PHP 生成器本质是受限字符表达式构造，先建字符索引表和递增优化。 | [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) |
| [D3CTF2021-virtual-love-revenge-2-0-wp](../raw/forensics/D3CTF2021-virtual-love-revenge-2-0-wp.md) | VMware 配置损坏、零宽字符字典和单用户模式取证组合，先修复虚拟机再进入磁盘证据链。 | [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md) |
| [D3CTF2022-badw3ter-wp](../raw/stego/D3CTF2022-badw3ter-wp.md) | 图片图层、黑底 QR 和隐写链组合，先分离图层/背景再恢复二维码或图像隐藏信息。 | [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md) |
| [D3CTF2022-ohhhh-spf-wp](../raw/pentest/D3CTF2022-ohhhh-spf-wp.md) | SPF/DNS 记录是主线，先查询 TXT/SPF 解析链并确认域名授权语义。 | [dns.md](dns.md) |
| [D3CTF2022-wannawacca-wp](../raw/malware/D3CTF2022-wannawacca-wp.md) | Windows 内存镜像、勒索样本和流量协议复现组合，先 dump 可疑进程和文件再还原加密链。 | [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md) |
| [D3CTF2023-d3casino-wp](../raw/blockchain/D3CTF2023-d3casino-wp.md) | 链上伪随机可预测但限制合约代码大小，先用 vanity 地址、minimal proxy 和 CREATE2 控制调用者。 | [blockchain-smart-contract-exploitation.md](blockchain-smart-contract-exploitation.md) |
| [D3CTF2023-d3craft-wp](../raw/reverse/D3CTF2023-d3craft-wp.md) | Minecraft/PaperMC 移动事件和数据包语义差异可绕过，先逆插件目标位置再脚本化状态移动。 | [game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md) |
| [D3CTF2023-d3gif-wp](../raw/stego/D3CTF2023-d3gif-wp.md) | GIF/PNG/QR 媒体层叠，先拆帧和图像通道，再恢复二维码或文本 artifact。 | [pdf-png-gif-and-text-stego.md](pdf-png-gif-and-text-stego.md) |
| [D3CTF2023-d3image-wp](../raw/forensics/D3CTF2023-d3image-wp.md) | 图像/二维码/摩斯或像素模式是主线，先分离颜色层、定位图案再恢复可读编码。 | [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md) |
| [D3CTF2023-d3readfile-wp](../raw/web/D3CTF2023-d3readfile-wp.md) | 任意文件读不知道 flag 路径时，先读取 locate 数据库并在本地搜索目标路径。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| [D3CTF2025-d3image-wp](../raw/ai-ml/D3CTF2025-d3image-wp.md) | 图像块变换和隐写编码可逆，先抽出块顺序/颜色关系再写正反向恢复脚本。 | [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md) |
| [D3CTF2025-d3rpg-signin-wp](../raw/stego/D3CTF2025-d3rpg-signin-wp.md) | RPG 地图、标牌、隐藏路径和摩斯地板构成视觉/空间隐写主线，先固定地图状态再提取场景线索。 | [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md)、[game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md) |
| [D3CTF2025-d3rpki-wp](../raw/pentest/D3CTF2025-d3rpki-wp.md) | RPKI/BGP 只校验 origin 时可伪造 peer 或宣告更具体路由，先确认 AS、CIDR 和 filter。 | [bgp-rpki-route-hijack.md](bgp-rpki-route-hijack.md) |

## 常见陷阱

- 把 WP 当完整题解目录使用，没有提取可迁移的触发信号。
- 只按题名命中下一跳，忽略源码、参数规模、可控输入和错误反馈。
- 已有技巧页能覆盖时仍新建单题页面，导致 wiki 退化成 writeup 列表。

## 关联技巧

- [bgp-rpki-route-hijack.md](bgp-rpki-route-hijack.md)
- [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)
- [dns.md](dns.md)
- [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md)
- [exotic-secret-sharing-rabin-and-polynomials.md](exotic-secret-sharing-rabin-and-polynomials.md)
- [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md)
- [font-shader-firmware-and-legacy-patterns.md](font-shader-firmware-and-legacy-patterns.md)
- [game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md)
- [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)
- [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md)
- [interactive-containers-jails-and-solvers.md](interactive-containers-jails-and-solvers.md)
- [keyboard-mouse-audio-and-physical-puzzles.md](keyboard-mouse-audio-and-physical-puzzles.md)
- [known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md)
- [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md)
- [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)
- [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md)
- [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md)
- [pdf-png-gif-and-text-stego.md](pdf-png-gif-and-text-stego.md)
- [pyjails.md](pyjails.md)
- [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)
- [scripts-and-obfuscation.md](scripts-and-obfuscation.md)
- [source-backdoors-and-restricted-shell-tricks.md](source-backdoors-and-restricted-shell-tricks.md)
- [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)
- [blockchain-smart-contract-exploitation.md](blockchain-smart-contract-exploitation.md)
- [windows-registry-logs-and-credentials.md](windows-registry-logs-and-credentials.md)
- [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md)

## 原始资料

本页原始资料以“WP 案例沉淀”表中的 raw WP 链接为准，共 57 篇。
