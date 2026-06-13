# CTF Wiki Log

本文件为 append-only 时间线。每个条目使用固定标题格式，便于 grep / 搜索。

## [2026-05-20] init | create ctf-wiki skeleton

- path: `D:/文档/markdown文件/ctf-wiki`
- created:
  - `AGENTS.md`
  - `CLAUDE.md`
  - `index.md`
  - `log.md`
  - `raw/`, `sources/`, `wiki/`, `sync/`, `templates/`, `assets/`
- decision:
  - 名称使用 `ctf-wiki`，不限定为 writeup；技术博客、论文、工具文档、靶场复盘也可以作为原始资料。
  - 本库是按需查询的补充层，不是 skill 默认上下文。

## [2026-05-21] raw-normalize | reverse PDF to Markdown

- category: `reverse`
- processed:
  - `raw/reverse/I'm a human WP.pdf` -> `raw/reverse/I'm a human WP.md`, images: `raw/reverse/I'm a human WP_images/`
  - `raw/reverse/Link Start! WP.pdf` -> `raw/reverse/Link Start! WP.md`, images: `raw/reverse/Link Start! WP_images/`
  - `raw/reverse/xxxtea WP.pdf` -> `raw/reverse/xxxtea WP.md`, images: `raw/reverse/xxxtea WP_images/`
- rule: PDF 原件不修改；Markdown 版同层存储；每个 PDF 的图片提取到独立同层目录，目录内只放图片。

## [2026-05-21] governance | raw format rules and skill sync planning

- updated: `AGENTS.md`
  - added Markdown/PDF raw source handling rules
  - added PDF-to-Markdown normalization convention
  - added per-PDF image extraction directory rule
- created: `sync/ctf-skill-reference-migration-plan.md`
  - proposed L1/L2 split between `ctf-* / references/` and `ctf-wiki`
  - listed A/B/C migration priority tiers
- updated external skill: `C:/Users/LMY/.agents/skills/ctf-writeup/SKILL.md`
  - generated writeups should also be archived to `ctf-wiki/raw/<category>/`
  - same-challenge old generated WP should be replaced, not duplicated

## [2026-05-21] migrate | raw-first archive of first skill references

- mode: raw-first migration
- policy: 原 skill reference 暂不修改；先归档全文快照，再生成 source summary。
- migrated:
  - `ctf-web/references/sql-injection.md` -> `raw/web/sqli-filter-and-oracle.md`
  - `ctf-crypto/references/aes-modes-mac-and-hash-oracles.md` -> `raw/crypto/aes-modes-mac-and-oracles.md`
  - `ctf-pwn/references/oob-jit-parser-and-readonly-primitives.md` -> `raw/pwn/oob-jit-parser-primitives.md`
  - `ctf-reverse/references/vm-obfuscation-and-transform-patterns.md` -> `raw/reverse/vm-obfuscation-transform-patterns.md`
  - `ctf-forensics/references/pcap-protocol-and-credential-recovery.md` -> `raw/forensics/pcap-protocol-credential-recovery.md`
- source summaries:
  - `sources/web/web-sqli-filter-and-oracle-skill-reference-source-summary.md`
  - `sources/crypto/crypto-aes-modes-mac-and-oracles-skill-reference-source-summary.md`
  - `sources/pwn/pwn-oob-jit-parser-primitives-skill-reference-source-summary.md`
  - `sources/reverse/reverse-vm-obfuscation-transform-patterns-skill-reference-source-summary.md`
  - `sources/forensics/forensics-pcap-protocol-credential-recovery-skill-reference-source-summary.md`
- next: 从这些 source summaries 提炼 family / technique 页面；之后再考虑给原 skill references 添加软链接。

## [2026-05-21] extract | family technique pages and skill reference slimming

- extracted family pages:
  - `wiki/families/web-sqli-filter-and-oracle-family.md`
  - `wiki/families/crypto-block-mode-misuse-family.md`
  - `wiki/families/pwn-oob-jit-parser-primitives-family.md`
  - `wiki/families/reverse-vm-obfuscation-transform-family.md`
  - `wiki/families/forensics-pcap-protocol-credential-recovery-family.md`
- extracted technique pages:
  - `wiki/techniques/web-sqli-filter-and-oracle-family.md`
  - `wiki/techniques/crypto-block-mode-misuse-family.md`
  - `wiki/techniques/crypto-block-mode-misuse-family.md`
  - `wiki/techniques/pwn-oob-jit-parser-primitives-family.md`
  - `wiki/techniques/reverse-vm-obfuscation-transform-family.md`
  - `wiki/techniques/forensics-pcap-protocol-credential-recovery-family.md`
- slimmed skill references:
  - `C:/Users/LMY/.agents/skills/ctf-web/references/sql-injection.md`
  - `C:/Users/LMY/.agents/skills/ctf-crypto/references/aes-modes-mac-and-hash-oracles.md`
  - `C:/Users/LMY/.agents/skills/ctf-pwn/references/oob-jit-parser-and-readonly-primitives.md`
  - `C:/Users/LMY/.agents/skills/ctf-reverse/references/vm-obfuscation-and-transform-patterns.md`
  - `C:/Users/LMY/.agents/skills/ctf-forensics/references/pcap-protocol-and-credential-recovery.md`
- policy: skill references now serve as L1 execution cards; `ctf-wiki` keeps family/technique/source/raw layers for on-demand expansion.

## [2026-05-21] migrate | full ctf skill reference coverage

- scope: all `ctf-* / references/*.md`
- processed:
  - raw/source/wiki records created or reused: 124
  - non-index reference files slimmed into L1 cards: 94
  - non-index reference files linked without additional slimming or already manually processed: 20
  - tooling references linked: 10
  - index references linked to category pages: 11
- policy:
  - `index.md` remains local routing index.
  - direction-specific tooling pages remain local machine/tool execution references.
  - non-index technique references now point upstream to `ctf-wiki`; large references are slimmed into L1 execution cards.

## [2026-05-21] validate | full ctf reference migration link check

- checked:
  - `ctf-* / references/*.md`: 135 files
  - `ctf-* / SKILL.md` plus references: 147 files
  - managed `ctf-wiki` markdown files: 285 files
- coverage:
  - `index.md` files with external wiki entry: 11 / 11
  - direction-specific tooling files with upstream wiki entry: 10 / 10
  - non-index/tooling reference files with upstream wiki entry: 114 / 114
- result: link / anchor / upstream absolute path / six-field-card validation passed with 0 errors.

## [2026-05-21] governance | backup location and flat raw category layout

- moved backup:
  - from `D:/文档/新建文件夹/ctf-reference-backup-20260521-113801`
  - to `D:/文档/markdown文件/ctf-wiki/backups/ctf-reference-backup-20260521-113801`
- updated: `AGENTS.md`
  - clarified that materials migrated from old skill references are ordinary category raw materials.
  - removed the special `raw/<category>/skill-references/` layout rule.

## [2026-05-21] migrate | flatten skill-reference raw and source files

- moved:
  - `raw/<category>/skill-references/*.md` -> `raw/<category>/*.md`
  - `sources/<category>/skill-references/*.md` -> `sources/<category>/*.md`
- removed empty `skill-references/` subdirectories under `raw/` and `sources/`.
- updated markdown links and upstream paths across skills and wiki.
- policy: do not preserve old skill reference locations as long-term metadata; maintain the new ctf-wiki structure directly.
- cleanup:
  - removed old `.md` relative links from migrated raw skill-reference materials, keeping the visible link text as ordinary text.
- validation:
  - checked `ctf-* / SKILL.md` plus references: 147 files
  - checked ctf-wiki markdown excluding backups: 409 files
  - remaining active `skill-references/` directories under `raw/` and `sources/`: 0
  - result: link / anchor / upstream absolute path / six-field-card validation passed with 0 errors.

## [2026-05-21] refine | raw layer as concrete material only

- decision:
  - `raw/` is material body, not navigation or abstraction layer.
  - standard six-field cards belong in `sources/`, `wiki/`, and `ctf-* / references/`, not in raw materials.
- updated: `AGENTS.md`
  - clarified that raw materials should keep concrete techniques, cases, payloads, commands, code snippets, challenge details, and derivations.
  - clarified that mode / signals / minimal evidence / solution skeleton / pitfalls / pivot should live above raw.
- cleaned:
  - removed standard `## 使用模式卡片` and six-field card paragraphs from 124 migrated raw skill-reference materials.
  - removed raw internal heading anchors from wiki/source links; wiki pages now link to raw files rather than using raw as a navigation layer.
- validation:
  - raw skill-reference files checked: 124
  - raw standard-card residue files: 0
  - checked `ctf-* / SKILL.md` plus references: 147 files
  - checked ctf-wiki markdown excluding backups: 409 files
  - result: link / anchor / upstream absolute path / six-field-card validation passed with 0 errors.

## 2026-05-21 — 撤回 solve-challenge 外接层并精简 raw 文件名

- 将 `ctf-solve-challenge` 重新定位为 skill 本地分流入口：撤回 active wiki 中的 `raw/solve-challenge/`、`sources/solve-challenge/`、`wiki/categories/solve-challenge.md` 和相关 technique 页面。
- 从历史备份恢复 `ctf-solve-challenge/references/` 的完整本地参考文件，并移除外接 wiki 依赖。
- 保留 `raw/<category>/` 分类目录，去掉从旧 skill reference 迁入 raw 文件名中的 category 前缀和 `-skill-reference` 后缀；本轮重命名 122 个 raw 文件。
- 本轮操作前的 active 对象备份位于：`D:/文档/markdown文件/ctf-wiki/backups/pre-solve-withdraw-raw-rename-20260521-161829`。

## 2026-05-21 — 扁平知识图谱重构

- 按用户确认，将 `ctf-wiki/wiki/` 重构为扁平技巧知识图谱：所有 active wiki 页面直接位于 `wiki/*.md`。
- 删除 active `sources/`、`sync/`、`assets/`、`scripts/`、`templates/`；不再维护来源视角摘要和同步候选缓冲区。
- 合并原工具子目录为统一工具入口页。
- 删除原 `wiki/categories/`、`wiki/techniques/`、`wiki/families/`、`wiki/tools/` 以及空的 `pivots/`、`patterns/`、`checklists/` 子目录。
- 更新 `ctf-* / references/` 的上游链接：保留 `wiki page` 与必要 `raw material`，移除 category/source-summary 链路。
- 本轮操作前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-flat-graph-restructure-20260521-191219`。

## 2026-05-21 — 删除专项 CTF skill references，改为 SKILL.md 直连 wiki

- 保留 `ctf-solve-challenge/references/`。
- 删除 10 个专项 CTF skill 的 active `references/` 目录：`ctf-ai-ml`、`ctf-crypto`、`ctf-forensics`、`ctf-malware`、`ctf-misc`、`ctf-osint`、`ctf-pentest`、`ctf-pwn`、`ctf-reverse`、`ctf-web`。
- 重写这些专项 `SKILL.md` 的参考入口，使其直接指向 `D:/文档/markdown文件/ctf-wiki/wiki/*.md` 和工具入口页。
- 本轮操作前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-remove-ctf-skill-references-20260521-194353`。

## 2026-05-21 — 摄入 KernelMaze / CythonChecker reverse PDF

- 在 `raw/reverse/` 中处理新增 PDF：`KernelMaze_WriteUp.pdf`、`CythonChecker_WriteUp.pdf`。
- 已生成同层 Markdown：`KernelMaze_WriteUp.md`、`CythonChecker_WriteUp.md`。
- 两份 PDF 均未发现嵌入图片，因此没有创建对应 `_images/` 目录。
- 新增技巧页：`wiki/windows-kernel-ioctl-hidden-feedback-maze.md`、`wiki/embedded-python-pyd-custom-aes.md`。
- 同步更新 `ctf-reverse/SKILL.md` 高频页面直链，并重建 `index.md`。

## 2026-05-21 — 第一轮吸收未接入 raw 到 wiki

- 处理此前未被 wiki frontmatter 引用的 9 个 raw 对象：3 组 reverse PDF/Markdown 与 3 个 web writeup。
- 新增技巧页：`powershell-staged-payload-and-clipboard-phishing.md`、`vmp-client-server-smc-rc4-recovery.md`、`compare-breakpoint-plaintext-recovery.md`、`path-confusion-to-signed-internal-request-chain.md`、`artifact-trust-ssrf-to-node-require-rce.md`、`json-duplicate-key-hmac-parser-differential.md`、`workflow-runner-internal-api-chain.md`。
- 对旧 PDF 派生 Markdown 清理摄入包装头，仅保留题解正文与图片引用。
- 同步更新 `ctf-reverse`、`ctf-malware`、`ctf-web` 的 `SKILL.md` 高频 wiki 直链。
- 重建 `index.md`，在 skill 视图和全量页面表中加入一行摘要与 raw 数量。

## 2026-05-21 — 全量 wiki 技巧页重组

- 将旧 `六字段卡片 / 主题索引` 目录页重写为标准技巧页结构：适用场景、识别信号、最小证据、解法骨架、关键变体、常见陷阱、关联技巧、原始资料。
- 删除 6 个无直接 raw 的薄页，并把内容与入口合并到下列目标页。
- 合并目标：`block-mode-misuse-family.md`、`vm-obfuscation-transform-family.md`、`oob-jit-parser-primitives-family.md`、`sqli-filter-and-oracle-family.md`、`pcap-protocol-credential-recovery-family.md`。
- 重写统一工具入口页，移除旧目录式内容。
- 恢复 KernelMaze、CythonChecker、PowerShell staged payload、VMP/SMC/RC4、compare breakpoint、三条 Web chain 等具体技巧页的专项内容。
- 重建 `index.md`，同步 skill 视图和全量页面表。
## 2026-05-21 — 工具页按方向拆分

- 将 raw 下各方向工具文件迁入 `wiki/<direction>-tooling.md`，使工具说明成为整理后的 wiki 知识页，而不是 raw 原始资料。
- 删除旧统一工具页，避免与各专项 `SKILL.md` 的首轮工具摘要重复。
- 同步更新所有专项 `ctf-* / SKILL.md`，使其直接指向对应方向工具页。
- 更新 `AGENTS.md`、`ctf-writeup/SKILL.md` 和 `index.md` 中的工具维护规则。

## 2026-05-21 — wiki 内容质量与工具页补强

- 清理 11 个真实 wiki 自链接，保留同名 raw 原始资料链接。
- 给 5 个 family 页补充具体技巧页回链，强化扁平知识图谱的技巧族入口。
- 精修 20 个较薄技巧页，重点补强适用场景、识别信号、最小证据、解法骨架、关键变体和常见陷阱。
- 在 `index.md` 增加“按题面信号查询”区域，使 agent 能从初始症状直接跳到具体技巧页。
- 统一 10 个方向工具页的结构，新增调用层、覆盖状态和后续补强方向，并规范详细清单层级。

## 2026-05-21 — raw 两类资料与 WP 归档规则

- 在 `AGENTS.md` 中补充 raw 两类资料规则：综述型原始资料与单题 WP 原始资料。
- 明确默认禁止改动已有 raw 正文；只有用户明确要求整理、PDF 处理、agent WP 替换或结构迁移确认时才允许修改 raw。
- 补充 family 页维护规则：family 是技巧族 pivot 入口，不是分类目录；仅在能连接多个具体 technique 并提供变体判断价值时保留。
- 更新 `ctf-writeup/SKILL.md`：单题 WP 必须包含 `Challenge Brief`、`Solution`、`Method Summary`，并带 `raw_type: writeup` 与 `source_origin: agent-generated`。
- 修正模板中的嵌套 Markdown code fence，避免示例模板提前闭合。

## 2026-05-21 — raw 分类改为文件名规则

- 按用户确认，取消 raw / WP 模板中的复杂 frontmatter 要求。
- raw 类型由文件名判断：包含 `wp` 或 `writeup` 的 raw 文件视为单题 WP；没有这些字样的 raw 文件默认视为综述型资料。
- 更新 `ctf-writeup/SKILL.md`：提交版和教学版模板不再包含 YAML frontmatter，保留 `Challenge Brief`、`Solution`、`Method Summary` 和 `Flag`。
- 更新同题旧版本检查逻辑：优先按文件名 slug、`wp/writeup` 语义和正文一级标题匹配，不再依赖 frontmatter。

## 2026-05-22 — wiki 查询工作流、family pivot 与工具一致性审校

- 在 `AGENTS.md` 中补充 Ingest / Query / Lint 三类运行工作流，明确新增 raw、按需查询和结构审校的操作顺序。
- 在 `index.md` 增加“按任务查询”和“按失败状态查询”，让 agent 能从任务目标或卡住状态直接跳到 wiki 页面。
- 给剩余孤立技巧页 `windows-kernel-ioctl-hidden-feedback-maze.md` 增加来自 `reverse-first-pass-workflow-and-debugging.md` 的有意义入链。
- 精修 5 个 family 页：把泛化变体表改为带优先证据、下一跳页面和失败后 pivot 的技巧族判断表。
- 校验 10 个方向工具页与 `ctf-* / SKILL.md` 的工具页指向和专项全路径工具覆盖情况。

## 2026-05-24 — ctf-writeup skill 结构重排与 wiki 沉淀规则迁移

- 按用户确认，重排 `ctf-writeup/SKILL.md`：先写适用边界、固定结构、总原则和模板，再写三段内容规则、附件规则、题型规则、PDF 拆分、已有 WP 审核、raw 归档和质量检查。
- 从 `ctf-writeup/SKILL.md` 移除 Reference / Wiki 沉淀规则，避免输出层 skill 承担知识库维护职责。
- 在 `AGENTS.md` 中补充“从 WP 沉淀到 wiki 的判断规则”，明确什么时候值得沉淀、什么时候不要沉淀、写到哪里和推荐沉淀格式。
- 将 `AGENTS.md` 中单题 WP 推荐结构统一为中文三段：`题目简述 / 解题过程 / 方法总结`。
- 继续精简 `ctf-writeup/SKILL.md`：把模板并入固定输出结构，把三段写法并入写作总原则，把不同题型材料规则并入附件与题目信息保留规则。

## 2026-05-31 — raw WP 图片目录同名化

- 将 5 个不符合“图片目录名与同级 WP basename 一致”规则的 raw 图片目录改名：
  - `raw/misc/ez_iotWP/` -> `raw/misc/VNCTF2026-ez-iot-wp/`
  - `raw/misc/HuntingAgentWP/` -> `raw/misc/VNCTF2026-huntingagent-wp/`
  - `raw/misc/MinecraftWP/` -> `raw/misc/VNCTF2026-minecraft-wp/`
  - `raw/misc/V(N)ShellWP/` -> `raw/misc/VNCTF2026-vnshell-wp/`
  - `raw/web/渗透测试WP/` -> `raw/web/VNCTF2026-web-pentest-wp/`
- 同步替换对应 5 个 WP Markdown 中的图片相对链接。
- 未调整 `index.md`：本轮只影响 raw 图片资源路径，不影响 wiki 查询入口。

## 2026-06-04 — 全量 raw WP 沉淀到 wiki

- 按用户确认，将 `raw/` 中现存 241 篇 WP 视为新的原始资料并沉淀到 wiki。
- 新建 pivot family 页：
  - `wiki/crypto-parameter-triage-family.md`：承载 43 篇 crypto WP，按 RSA、ECC/DLP、格、PRNG、分组模式、哈希协议、同态/代数等下一跳分流。
  - `wiki/misc-cross-category-triage-family.md`：承载 35 篇 misc WP，按编码链、游戏状态、restricted shell、pyjail、oracle/递推、音频/物理和文件首检分流。
- 在既有入口页追加 `WP 案例沉淀` 表：
  - `wiki/ml-model-inference-extraction-and-weight-analysis.md`：6 篇 AI/ML WP。
  - `wiki/cross-domain-forensics-technique-map.md`：2 篇 forensics WP。
  - `wiki/geolocation-and-media.md`：2 篇 osint WP。
  - `wiki/pwn-first-pass-red-flags-and-protections.md`：38 篇 pwn WP。
  - `wiki/reverse-first-pass-workflow-and-debugging.md`：66 篇 reverse WP。
  - `wiki/web-first-pass-triage-and-chain-patterns.md`：49 篇 web WP。
- 每条 WP 只抽取 raw 链接、可复用识别信号和下一跳技巧页；没有把单题流水账搬进 wiki。
- 更新 `index.md`：新增 crypto/misc pivot 查询入口、更新 raw 统计、同步新增页和承载页 raw 计数。
- 同步更新 `ctf-crypto/SKILL.md` 与 `ctf-misc/SKILL.md` 的高频 wiki 直链，指向新增 pivot 页。
- 本轮操作前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-wp-full-ingest-20260604-101000`。

## 2026-06-04 — raw/wiki 健康检查与案例映射复核

- 对 active tree、`wiki/` 扁平结构、Markdown 链接、frontmatter raw 路径、raw 覆盖率和 WP 案例表做全量校验。
- 校验结果：`wiki/*.md` 仍为 133 个扁平 Markdown 页面；检查 1713 个本地 Markdown 链接，断链 0；353 个 raw Markdown 均被 wiki 引用；241 篇 raw WP 均出现在 `WP 案例沉淀` 表且无重复/缺失。
- 内容复核中重点抽查 26 条 raw->wiki 高风险映射，修正 misc/pwn/reverse/web 首轮承载页中的 26 条下一跳与复用信号。
- 主要修正方向：把 misc 中的 dosemu2/容器逃逸和弱 RSA 参数分别 pivot 到交互容器与 crypto/RSA；把 pwn 中的 heap/UAF、格式串、FSOP、runtime primitive、Web+firmware chain 等从宽泛首轮页落到具体技巧页；把 reverse 中的 Rust/GMP、eBPF、UPX、PyInstaller、文件伪装和协议解析落到更合适的技巧页；把 web 中的 JPHP、上传 WebShell、Pickle/PyTorch 与 Java 反序列化从 SQL/XSS 误挂改到对应技巧页。
- 同步补充 `reverse-first-pass-workflow-and-debugging.md` 与 `web-first-pass-triage-and-chain-patterns.md` 的关联技巧入口，并更新 `misc-cross-category-triage-family.md` 的变体计数。
- 本轮未改写 raw 正文；修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-health-audit-fix-20260604-104142`。

## 2026-06-11 — D3CTF raw WP 沉淀到 wiki

- 按用户确认，将 `raw/` 中现存 130 篇 D3CTF WP 视为新的原始资料并沉淀到 wiki。
- 按方向写入既有首轮承载页：
  - `wiki/crypto-parameter-triage-family.md`：新增 21 篇 D3CTF crypto WP，补充分组、ECC、secret sharing、哈希、同态、格、PRNG、数论和 RSA 小根/CRT 分流。
  - `wiki/misc-cross-category-triage-family.md`：新增 20 篇 D3CTF misc WP，扩展到 Web3、字体资源、Chrome/DPAPI、VM/内存取证、PCAP、图像隐写、DNS/SPF、任意读路径推断和 RPKI/BGP。
  - `wiki/pwn-first-pass-red-flags-and-protections.md`：新增 28 篇 D3CTF pwn WP，覆盖 heap/UAF、FSOP、format string、QEMU/ESXi/TEE、Chrome/V8、kernel/eBPF/FUSE、OpenWrt/AArch64 和 FastCGI CVE。
  - `wiki/reverse-first-pass-workflow-and-debugging.md`：新增 30 篇 D3CTF reverse WP，覆盖 VM/OISC、文件系统镜像、compressed memory、固件/串口、SIMD、Android/Unity、Cython、WoW64、syscall VM、Windows driver 和 VMP。
  - `wiki/web-first-pass-triage-and-chain-patterns.md`：新增 31 篇 D3CTF web WP，覆盖 XSS/CSRF、SQLi+RCE、Node prototype pollution、PHP/Java/Python 反序列化、云/Serverless、NoSQL、WordPress/CVE、JWT/MinIO、jtar 和 QUIC parser 差异。
- 新建 `wiki/bgp-rpki-route-hijack.md`，承接 `D3CTF2025-d3rpki-wp` 中 RPKI origin-only 校验、BGP peer 伪造和更具体前缀宣告的可复用技巧。
- 更新 `index.md`：新增 RPKI/BGP 题面信号入口、更新 raw 统计、同步承载页 raw 计数和新增页面索引。
- 校验结果：`wiki/*.md` 为 134 个扁平 Markdown 页面；检查 2015 个本地 Markdown 链接，断链 0；483 个 raw Markdown 均被 wiki 覆盖；371 篇 raw WP 均出现在 `WP 案例沉淀` 表且无重复/缺失；130 篇 D3CTF WP 均已被 wiki 引用。
- 本轮未改写 raw 正文；修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-d3ctf-ingest-20260611-152634`。

## 2026-06-11 — raw/wiki 健康检查与 case 映射修正

- 对 active tree、wiki 扁平结构、raw 覆盖率、frontmatter raw 路径、WP case 表和 D3CTF 映射做复核。
- 机械校验确认：483 个 raw Markdown 均被 wiki 覆盖；371 篇 raw WP 均进入 `WP 案例沉淀` 表；130 篇 D3CTF WP 均有 wiki 引用和 case 入口。
- 内容复核后修正 30 条 raw->wiki case 映射：把 Solana/Sui/LLM/OSINT/ZipCrypto/TOCTOU/C2 等 misc 误挂归入 blockchain-smart-contract-exploitation、llm、social-media、disk-recovery、race、c2 等页面；把 pwn 中 CPU 侧信道、MCU 格式串、FSOP、largebin、整数溢出落到具体技巧页；把 reverse 中 loader+eBPF+kernel、Java/JAR、Windows driver、Android zygote、SPARC ROM、compare-breakpoint 等落到更准确的下一跳；把 web 中 PHP include 和 Redis race 从 auth/SSRF 误挂改到对应技巧页。
- 同步更新 `misc-cross-category-triage-family.md` 的变体计数；本轮未改写 raw 正文。
- 另发现 `raw/ai-ml/SU_theifWP.md` 原文引用 `./app.py`，但 raw 中没有该附件；因 raw 正文默认只读，本轮仅记录为原始资料缺口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-health-audit-fix-20260611-155909`。

## 2026-06-12 — WMCTF2025 raw WP 摄入

- 按 Ingest 流程处理 raw 中新增的 21 篇 WMCTF2025 WP；本轮不改写 raw 正文，只把可复用识别信号、最小证据和解法骨架沉淀到 active wiki。
- 新增 `wiki/protocol-relay-and-internal-service-injection.md`：承载 RustDesk `RequestRelay` 这类协议转发到本机/内网服务并造成 Redis 命令注入的模式，避免硬塞进普通 HTTP SSRF 或路径穿越页。
- 更新五个方向入口：
  - Web：`guess`、`pdf2text`、`Rustdesk change client backend` 分别路由到 MT19937 预测/Python eval 逃逸、pdfminer CMap pickle、协议 relay 到 Redis。
  - Crypto：`IShowSplit`、`SplitMaster`、`LW3`、`LW5` 归入 lattice/LWE，`LemonPepper` 归入 p-adic/代数恢复。
  - Misc/Forensics：`GitHacker` 归入 Git reflog + VeraCrypt 卷头恢复，`Voice_hacker` 归入 RTP 音频恢复和 wav 特征伪造，`Shopping company & phishing email` 归入 LLM 工具调用、Open WebUI N-day 和 SVG/JS 附件分析。
  - Pwn：`Aberration`、`PaluSimulator`、`wm_easyker`、`wm_easynetlink`、`wm_eat_some_qanux`、`wmkpf` 分别沉淀到 TrustZone/竞态、C++ 堆链、Linux kernel ROP/eBPF/页表和自定义 VM syscall 路线。
  - Reverse：`appfriend`、`catfriend`、`VideoPlayer`、`Want2BecomeMagicalGirl` 分别沉淀到 Android native crypto、Mach-O 反调试 + 魔改 ChaCha、Windows VMP 动态 dump 和 Flutter/Java/native 反 hook 执行流。
- 同步 `index.md`：新增 Web technique 入口，Raw 统计更新为 crypto 84、misc 71、pwn 92、reverse 117、web 102。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-wmctf2025-ingest-20260612-145123.zip`。

## 2026-06-11 — technique 页反向案例索引补强

- 针对 raw WP 只在方向首轮承载页中出现、下一跳 technique 页缺少反向案例入口的问题，补强 10 个 technique 页的 `来自 WP 的案例索引`。
- 补强页面：
  - `blockchain-smart-contract-exploitation.md`：Solana direct mapping、Anchor/PDA、Sui Move capability、以太坊私钥/伪随机/CREATE2 案例。
  - `race-condition-and-concurrency-exploits.md`：生产者消费者并发、SUID TOCTOU、Redis 订单/退款 lock key 竞争。
  - `malware-c2-session-key-and-protocol-recovery.md`：Merlin Agent dump 与 OPAQUE 会话密钥解密流量。
  - `osint-account-public-media-correlation.md`：博客/GitHub/邮箱/Minecraft/Discord/CTFtime/gopher/公开媒体 OSINT 链。
  - `filesystem-archive-recovery-and-repair.md`：ZipCrypto + AVIF/ISOBMFF 已知明文恢复。
  - `llm-attacks.md`：隐藏 prompt/历史对话泄露与 LLM 约束绕过。
  - `hardware-isa-bootloader-and-kvm.md`：RISC-V CPU side channel、MCU/UART/I2C、SPARC ROM、STM32 dongle、SIMD lane 排列。
  - `go-rust-jvm-and-cpp-reversing.md`：Spring Boot/JAR、Java 游戏、Rust/GMP、Cython/Python 扩展。
  - `windows-kernel-ioctl-hidden-feedback-maze.md`：Windows IOCTL nonce/HMAC/blob、R3/R0 反调试、WoW64 隐藏逻辑。
  - `compare-breakpoint-plaintext-recovery.md`：复杂加密外观下最终比较点明文恢复。
- 更新 `index.md` 的题面信号入口和 Raw 计数，让这些技术页能从题面直接命中，并保持索引统计与页面 raw 链接一致。
- 本轮未改写 raw 正文；修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-structure-content-tune-20260611-162602`。

## 2026-06-11 — 弱命名 technique 页手工重构

- 按用户要求，不使用脚本批量修改；逐页阅读并定点重构弱命名页面，目标是让文件名、标题、适用场景和 raw 案例的真实技术模式一致。
- 已授权修复 raw 缺失附件链接：`raw/ai-ml/SU_theifWP.md` 中原本的 `app.py` 相对链接改为普通附件名描述，避免指向不存在的同层文件。
- 重命名并重写 5 个弱 technique 节点：
  - `wiki/web3.md` -> `wiki/blockchain-smart-contract-exploitation.md`：聚焦 EVM/Solana/Anchor/Sui Move、链上伪随机、CREATE2、账户/capability 绑定和交易状态。
  - `wiki/social-media.md` -> `wiki/osint-account-public-media-correlation.md`：聚焦公开账号、媒体、commit、平台昵称、历史快照和身份链证据闭合。
  - `wiki/c2-and-protocols.md` -> `wiki/malware-c2-session-key-and-protocol-recovery.md`：聚焦样本/内存/配置/PCAP 组合下的会话 key 与协议恢复。
  - `wiki/model-attacks.md` -> `wiki/ml-model-inference-extraction-and-weight-analysis.md`：聚焦模型推理、抽取、反演、权重/adapter 和任务语义恢复。
  - `wiki/disk-recovery.md` -> `wiki/filesystem-archive-recovery-and-repair.md`：聚焦文件系统、压缩包修复、删除文件、嵌套容器和已知明文恢复。
- 手工更新 `index.md`、相关 family/technique 页和相邻技巧入链，保留 raw 文件名作为原始资料引用，不把 raw 名称强行改成 wiki 名称。
- 在 `index.md` 新增 `核心知识网络`，把首轮分流、状态机与协议、文件与证据恢复、运行时与逆向、链上/AI/公开情报五类节点串成查询入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-manual-network-retune-20260611-raw-web3`。

## 2026-06-11 — index 分类收敛与 family 类型统一

- 按用户确认，将 `wiki/` 页面类型收敛为 `family` / `technique` / `tooling` 三类。
- 将首轮/方向/地图类页面统一视为 `family`：
  - `pwn-first-pass-red-flags-and-protections.md`
  - `reverse-first-pass-workflow-and-debugging.md`
  - `web-first-pass-triage-and-chain-patterns.md`
  - `cross-domain-forensics-technique-map.md`
  - `cross-primitive-escape-and-hybrid-exploit-map.md`
  - `file-triage-archives-and-one-liners.md`
- 重写 `index.md`：删除冗长的按题面信号和失败状态大表，改为页面类型说明、family 索引、tooling 索引、按方向分类的 technique 索引、raw 统计和维护入口。
- 校验目标：所有 `wiki/*.md` 都必须属于三类之一；`index.md` 覆盖所有 wiki 页面；root/wiki 本地 Markdown 链接无断链。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-index-family-retune-20260611`。

## 2026-06-11 — 维护规则同步语义健康检查

- 更新 `AGENTS.md`，将 `wiki/` active 页面类型明确收敛为 `family` / `technique` / `tooling` 三类。
- 明确首轮/方向页、triage 页和跨技巧 map 统一视为 `family`，family 负责连接 technique、提供首轮判断、变体 pivot 和失败后转向。
- 将维护流程拆成三层：
  - `Ingest`：把 raw 资料提炼进 wiki；
  - `Structural Lint`：每次结构性修改后检查三类 type、扁平结构、链接、index 覆盖、raw 引用和 log；
  - `Semantic Health Check`：仅在用户明确要求健康检查或重组知识网络时执行，从 technique 页开始复核 raw 归属、合并/拆分/删除，再更新 family 和 index。
- 更新 `index.md` 职责说明：index 保持轻量分类索引，题面信号、失败状态和细粒度 pivot 应主要写在 family 或 technique 页中。
- 本轮只同步维护规则，未启动 technique 语义健康检查。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-agents-rule-update-20260611`。

## 2026-06-11 — backup zip 规则与健康检查启动

- 更新 `AGENTS.md`：`backups/` 下新增和保留的备份必须为 `.zip` 压缩包；若临时展开备份用于核对，核对完成后必须删除展开目录。
- 将历史备份目录压缩为 zip，并在验证 zip 可打开后删除展开目录；迁移后 `backups/` 中无目录、无非 zip 文件。
- 健康检查第一轮从 technique 页开始，先筛查工具化命名和模板化内容：
  - `disassemblers-debuggers-and-basic-tools.md`
  - `frida-angr-lldb-and-x64dbg.md`
  - `qiling-triton-pin-and-ldpreload.md`
- 上述 3 页内容主要是工具入口、调试/插桩/模拟调用边界和安装/命令，而不是具体攻击技巧，已从 `type: technique` 改为 `type: tooling`，并从 index 的 Reverse technique 列表移入 tooling 列表。
- 保留 `kaslr-kpti-smep-and-kernel-debugging.md` 为 technique：虽然名称含 debugging，但主题是 KASLR/KPTI/SMEP 等内核利用保护绕过。
- 暂保留 `packers-deobfuscation-and-debug-automation.md` 为 technique：主题仍是壳识别、反混淆和调试自动化工作流；后续健康检查应优先重写其模板化内容。
- 修复前备份：
  - `D:/文档/markdown文件/ctf-wiki/backups/pre-backup-zip-rule-and-healthcheck-20260611-204947.zip`
  - `D:/文档/markdown文件/ctf-wiki/backups/pre-healthcheck-tooling-reclass-20260611-205321.zip`

## 2026-06-11 — 重写保留的高风险 technique 页

- 立即重写上一轮健康检查中暂保留的两个 technique 页，去掉模板化占位内容并按 raw 资料重建可复用技术模式：
  - `kaslr-kpti-smep-and-kernel-debugging.md`：聚焦 Linux kernel pwn 中 KASLR/FGKASLR、KPTI、SMEP/SMAP、`modprobe_path` / `core_pattern`、符号偏移恢复、QEMU/initramfs 调试和 exploit delivery 的路线选择。
  - `packers-deobfuscation-and-debug-automation.md`：聚焦壳/虚拟化/混淆样本的识别、动态 trace、差分、IR lifting、patch、约束提取和运行时 dump；工具入口改为指向 reverse tooling 相关页面。
- 两页均保留 `type: technique`，因为它们记录的是攻击/恢复工作流，而不是单纯工具清单。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-rewrite-kaslr-packers-techniques-20260611-210744.zip`。

## 2026-06-11 — tooling 页模板化复用重点修复

- 修复用户指出的 tooling 页重复机械文本问题：`disassemblers-debuggers-and-basic-tools.md`、`frida-angr-lldb-and-x64dbg.md`、`qiling-triton-pin-and-ldpreload.md` 不再使用 technique 模板中的“关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式”占位句。
- 三页保持 `type: tooling`，重写为工具路由页：
  - 基础反汇编/调试/格式工具页说明 GDB、r2、Ghidra、Unicorn、pyc/WASM/APK/.NET/packer 工具的使用边界和失败转向。
  - Frida/angr/lldb/x64dbg 页说明运行时 hook、符号执行和平台调试的适用条件。
  - Qiling/Triton/Pin/LD_PRELOAD 页说明完整模拟、动态符号执行、计数侧信道和 API 劫持的边界。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-tooling-pages-rewrite-20260611-211528.zip`。

## 2026-06-11 — Web technique 健康检查与 family 重分层

- 继续从 technique 页做语义健康检查，先处理 Web 方向中模板化程度最高且职责混杂的一簇页面。
- 将 5 个原 `type: technique` 页面改为 `type: family`，因为它们都承担多技术分流和 pivot，而不是单一可复用技巧：
  - `auth-bypass-cookies-and-hidden-routes.md`：认证、session、隐藏路由、代理 ACL、OAuth 和访问控制分流。
  - `php-lfi-ssti-ssrf-and-type-juggling.md`：PHP/LFI/SSTI/SSRF/XXE/type juggling 等服务端解释层差异分流。
  - `path-traversal-ssrf-upload-and-rsc.md`：路径穿越、上传、SSRF、渲染器、RSC 和内部服务链分流。
  - `ruby-php-upload-and-ssti-rce.md`：语言 eval、模板、上传、反序列化和命令执行链分流。
  - `xss-dom-and-browser-tricks.md`：XSS、DOM、admin bot、缓存/MIME、CSP 和浏览器外带分流。
- 未物理合并或删除上述页面：raw 内容覆盖面大但仍有明确首轮分流价值；直接删除会丢失现有 WP 案例入口。
- 暂不强制重命名为 `*-family.md`：当前文件名已经能概括主题，且全库入链较多；本轮优先修复类型和正文语义，避免大范围机械改链。
- 保留并重写 `csp-xsleak-and-browser-exfiltration.md` 为具体 `technique`：其核心是受 CSP/同源/无回显限制时构造浏览器 oracle 外带数据。
- 重写 `web-first-pass-triage-and-chain-patterns.md` 的表头分流逻辑，并保留后续 WP 案例沉淀表。
- 同步更新 `index.md`，将上述 5 页从 Technique 索引移动到 Family 索引。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-web-family-technique-retune-20260611-211927.zip`。

## 2026-06-11 — Web CVE 与 RCE 链路页 family 化

- 继续处理 Web 方向同簇高风险 technique，额外将 2 个原 `type: technique` 页面改为 `type: family`：
  - `known-cves-and-n-day-exploits.md`：CVE/N-day 是发现入口和版本边界判断，不是单一漏洞技术；页面改为按组件、客户端/服务端执行者、PoC 前置条件和后续链路分流。
  - `sqli-upload-deser-and-command-rce.md`：页面覆盖 SQLi、上传、反序列化、命令包装器、文件读和业务竞态到 RCE 的升级链，职责是链路分流而不是单点 technique。
- 这两页未删除、未物理合并：其 raw 仍承载短案例集合和链路速查价值，且已有多处 WP 下一跳引用。
- 同步更新 `index.md`，将两页移入 Family 索引；`web-first-pass-triage-and-chain-patterns.md` 已在上一条中改为引用这些 family。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-web-cve-rce-family-retune-20260611-212725.zip`。

## 2026-06-11 — tooling 页维护规则收紧

- 更新 `AGENTS.md` 的 tooling 页规则：工具页必须写具体工具路由、环境边界和失败转向，不得用可套用到任意页面的“关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式”占位句。
- 这条规则用于防止本轮修复过的 tooling 页重新退化成 raw 标题展开表或工具百科表。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-agents-tooling-rule-tighten-20260611-213054.zip`。

## 2026-06-12 — Reverse 高风险 technique family 化

- 继续执行语义健康检查，处理 Reverse 方向中模板化程度最高、且标题明显承载多平台/多载体/多语言分流职责的一簇页面。
- 将 6 个原 `type: technique` 页面改为 `type: family`：
  - `android-games-hardware-and-runtime-platforms.md`：Android/APK、Flutter、Unity/Godot/Roblox、Electron/Node 和硬件抽象等运行时平台分流。
  - `mobile-firmware-kernel-and-game-re.md`：Mach-O/iOS、固件、驱动、eBPF、游戏引擎和 CAN/车载环境分流。
  - `go-rust-jvm-and-cpp-reversing.md`：Go、Rust、JVM/Kotlin、C++、Swift、D、Haskell、Cython/Nuitka 等语言运行时分流。
  - `python-bytecode-esolangs-and-uefi.md`：Python 字节码、opcode remap、Pyarmor/Nuitka、Brainfuck/FRACTRAN、HarmonyOS ABC、UEFI 等字节码/低频格式分流。
  - `loader-vm-image-and-kernel-patterns.md`：二阶段 loader、image/bitmap、kernel module、binfmt、shared library backdoor 和特殊执行链分流。
  - `font-shader-firmware-and-legacy-patterns.md`：字体、shader、BPF、MBR、legacy 格式、side-channel 和约束化恢复分流。
- 未删除或物理合并上述页面：它们仍有 raw 证据和 WP 下一跳价值；但不再占用具体 technique 名额。
- 暂不拆成更细 technique：当前许多分支仍是短案例速查，优先保留为 family，后续只有稳定高频模式积累足够 raw/WP 证据时再拆具体技巧页。
- 重写 `reverse-first-pass-workflow-and-debugging.md` 表头路由，使其成为 Reverse 总入口，并保留后续 WP 案例沉淀表。
- 同步更新 `index.md`，将上述 6 页从 Reverse Technique 索引移动到 Family 索引。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-reverse-family-retune-20260612-155311.zip`。

## 2026-06-12 — Crypto 高风险 technique family 化

- 继续执行语义健康检查，处理 Crypto 方向中模板化程度最高、且明显承载多攻击路线分流职责的一簇页面。
- 将 4 个原 `type: technique` 页面改为 `type: family`：
  - `rsa-attacks.md`：RSA 基础路线分流，覆盖小指数、共模、广播、Wiener、Fermat、多素数和基础 oracle。
  - `rsa-specialized-structures-and-oracles.md`：RSA 长尾结构分流，覆盖特殊素数、CRT 参数泄露、fault、ROCA、同态签名、Coppersmith 和复杂 oracle。
  - `hash-protocol-and-oracle-attacks.md`：Hash/MAC/协议 oracle 分流，覆盖 length extension、CRC/线性 MAC、compression oracle、padding/parse/timing oracle、SRP/DH proof 和状态可逆。
  - `classical-xor-and-substitution-ciphers.md`：古典替换、Vigenere、XOR、many-time pad、文件头推 key 和视觉轻量 cipher 分流。
- 未合并 `rsa-attacks.md` 与 `rsa-specialized-structures-and-oracles.md`：前者用于 RSA 快速首检，后者用于需要建模、oracle 或实现 bug 的长尾结构。
- 暂不拆小 e、共模、length extension、CRC、Vigenere、XOR 等小页：当前 raw 多为短案例速查，先保留在 family 中；后续有多篇 WP 支撑时再拆具体 technique。
- 同步更新 `index.md`，将上述 4 页从 Crypto Technique 索引移动到 Family 索引。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-crypto-family-retune-20260612-160050.zip`。

## 2026-06-12 — Pwn/Pentest 高风险 technique family 化

- 继续从 technique 页执行语义健康检查，处理 Pwn/Pentest 中模板化程度高且职责明显跨多个技巧路线的页面。
- 将 4 个原 `type: technique` 页面改为 `type: family`：
  - `overflow-basics.md`：栈溢出、ret2win、canary 泄露/爆破、协议长度越界、signed index、单字节覆盖和结构体指针覆盖的二级分流。
  - `linux-kernel-exploit-basics.md`：Linux kernel pwn 环境、保护组合、对象 primitive、eBPF/页表和提权目标分流，并保留 WMCTF kernel WP 案例索引。
  - `pentest-attack-chains-and-tunneling.md`：foothold、凭据、隧道、横向和提权 attack graph 分流。
  - `linux-privesc.md`：Linux 单机提权、服务配置滥用、数据库/备份凭据、NFS/SSH socket、本机代理和 race 路线分流。
- 未合并 `pentest-attack-chains-and-tunneling.md` 与 `linux-privesc.md`：前者负责跨主机链路建模，后者负责单台 Linux 上从低权到高权的路线族。
- 未把 `overflow-basics.md` 并入 `pwn-first-pass-red-flags-and-protections.md`：首轮页负责全 Pwn 漏洞族选择，overflow 页负责基础覆盖/OOB/canary 的二级分流。
- 同步更新 `index.md`，将上述 4 页从 Technique 索引移动到 Family 索引。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-pwn-pentest-family-retune-20260612-160621.zip`。

## 2026-06-12 — Forensics 证据源 technique family 化

- 继续执行语义健康检查，处理 Forensics 方向中模板化程度高、且明显承担证据源二级分流职责的一簇页面。
- 将 3 个原 `type: technique` 页面改为 `type: family`：
  - `windows-registry-logs-and-credentials.md`：Windows 注册表、事件日志、SAM/SYSTEM、NTFS、USN、浏览器凭据、Defender/WMI 和反取证时间线分流。
  - `pdf-png-gif-and-text-stego.md`：PDF、PNG/APNG、GIF、SVG、终端文本、表格、视频容器和文件叠加隐写分流。
  - `linux-git-browser-and-container-forensics.md`：Linux 日志、Git 对象库、浏览器 profile、Docker layer/history、KeePass 和运行中进程证据源分流。
- 未并入 `cross-domain-forensics-technique-map.md`：总 map 负责跨介质下一跳，这三页负责进入具体证据源后的二级分流。
- 暂不拆成 Windows EventLog/Registry/Browser、PDF/PNG/GIF、Git/Docker/Browser 等小页：当前 raw 多为同题链路或同证据源集合，拆分会导致入口碎片化；后续单一路线积累足够 raw/WP 证据时再拆具体 technique。
- 同步更新 `index.md`，将上述 3 页从 Forensics Technique 索引移动到 Family 索引。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-forensics-family-retune-20260612-161333.zip`。

## 2026-06-12 — Misc/OSINT 高风险 technique family 化

- 继续执行语义健康检查，处理 Misc/OSINT 中模板化程度高或明显承担分流职责的一簇页面。
- 将 6 个原 `type: technique` 页面改为 `type: family`：
  - `encodings-qr-and-esolangs.md`：Base/hex/URL/ROT、Unicode、二维码、esolang、浮点/数字列和多层编码链分流。
  - `pyjails.md`：Python 受限执行、对象链、字符集限制、装饰器/f-string、oracle、audit hook 和 agent sandbox 分流。
  - `vm-z3-sandbox-and-game-basics.md`：VM/解释器、WASM/游戏、PyInstaller/marshal、Z3/约束、K8s、浮点和 sandbox 分流。
  - `geolocation-and-media.md`：图片、视频、坐标、街景、路牌、地标、IP 和公开媒体地理定位分流。
  - `web-and-dns.md`：搜索引擎、公开文档、DNS、WHOIS、Wayback、Shodan、公开仓库和目录泄露分流。
  - `osint-account-public-media-correlation.md`：账号、公开媒体、游戏平台、社交平台、archive、GitHub/博客和身份链分流。
- 修正 `vm-z3-sandbox-and-game-basics.md` 在 `index.md` 中落入 Reverse Technique 区的错位：该页当前职责是 Misc 规则系统 family，复杂 VM/字节码才通过链接 pivot 到 Reverse 页面。
- 未物理合并 Misc/OSINT 页面：这些页面分别承担编码/QR、Python jail、规则系统、地理媒体、Web/DNS、账号身份链的二级分流，直接合并会让 family 失去 pivot 价值。
- 同步更新 `index.md`，将上述 6 页从 Technique 索引移动到 Family 索引；OSINT 当前 active 页面均为 family/tooling，暂无线性 technique 列表。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-misc-osint-family-retune-20260612-161803.zip`。

## 2026-06-12 — Pwn runtime/control-flow technique family 化

- 继续执行 technique 优先的语义健康检查，处理 Pwn 中模板化程度高、且明显承担二级分流职责的一簇页面。
- 将 5 个原 `type: technique` 页面改为 `type: family`：
  - `seccomp-ret2dlresolve-and-runtime-primitives.md`：Seccomp、ORW、ret2dlresolve、runtime syscall、GOT/PLT 和动态解析路线分流。
  - `interpreter-jit-canary-and-integer-exploits.md`：解释器、JIT、自定义 VM、整数宽度、canary、GC 和 timing oracle 的运行时 primitive 分流。
  - `runtime-protection-and-tls-exploits.md`：FILE/stdout、TLS destructor、atexit PTR_MANGLE、shadow stack、safe-linking 和 runtime callback 分流。
  - `stack-pivots-srop-and-seccomp-rop.md`：栈迁移、SROP、RETF、vDSO/vsyscall、JIT-ROP、ret2dlresolve 和 constrained ROP 分流。
  - `windows-arm-and-cross-platform-exploits.md`：Windows、ARM/Thumb/ARM64、DOS/m68k、Forth、SEH/CFG 和平台约束分流。
- 未删除或物理合并上述页面：它们都有 raw 支撑，并能在 Pwn 首轮页之后提供明确 pivot；直接合并到 `pwn-first-pass-red-flags-and-protections.md` 会让总入口过宽。
- 暂不拆出更多小 technique：当前 raw 多为短案例集合，先保留路线族判断；后续只有单一路线积累足够多 raw/WP 证据时再拆具体技巧页。
- 同步更新 `index.md`，将上述 5 页从 Pwn Technique 索引移动到 Family 索引。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-pwn-runtime-family-retune-20260612-162441.zip`。

## 2026-06-12 — Reverse/Malware 反分析与载荷链 family 化

- 继续执行 technique 优先的语义健康检查，处理仍含大量模板化变体表、且职责明显跨多个路线的 Reverse/Malware 页面。
- 将 3 个原 `type: technique` 页面改为 `type: family`：
  - `anti-analysis.md`：反调试、anti-VM、anti-sandbox、anti-DBI、自校验、反反汇编和移动端 hook 检测分流。
  - `scripts-and-obfuscation.md`：JS/PowerShell/SVG/邮件附件/包载荷、脚本混淆、C2 配置和恶意载荷链分流。
  - `self-decrypting-strings-and-lattice-patterns.md`：多层自解密、字符串恢复、prefix hash、格/线性约束、决策树和魔改 cipher 恢复分流。
- 未删除上述页面：它们均有 raw 和 WP 案例支撑，且在对应首轮页之后承担明确二级 pivot。
- 暂不拆成更细 technique：当前 raw 多为路线集合和短案例，拆分会造成入口碎片化；后续可在 Linux ptrace、Android Frida 检测、TLS/壳 dump、格约束恢复等方向积累足够 raw 后再拆。
- 同步更新 `index.md`，将上述 3 页从 Technique 索引移动到 Family 索引。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-reverse-malware-family-retune-20260612-163204.zip`。

## 2026-06-12 — Misc jail/format/interactive 页面 family 化

- 继续执行 technique 优先的语义健康检查，处理 Misc 中仍含模板化变体表且承担二级分流职责的页面。
- 将 3 个原 `type: technique` 页面改为 `type: family`：
  - `interactive-containers-jails-and-solvers.md`：交互服务、容器边界、jail 变体、虚拟化 helper、oracle 和小型求解器分流。
  - `bashjails.md`：bash/rbash 受限 shell、eval 上下文、字符集构造、输出通道和 post-shell 枚举分流。
  - `exotic-encodings-and-file-formats.md`：Verilog/HDL、Gray code、RTF/SMS PDU/UTF-9、MaxiCode、DTMF、音乐音程和解析器差异等低频格式分流。
- 未合并 `exotic-encodings-and-file-formats.md` 到 `encodings-qr-and-esolangs.md`：后者负责轻量可逆编码和 QR/esolang 首轮判断，前者负责需要格式规范、字段清洗、信号映射或解析器语义的低频格式。
- 未合并 `bashjails.md` 到 `pyjails.md`：shell jail 的 eval、fd、变量和输出通道模型与 Python 对象/namespace 模型不同。
- 同步更新 `index.md`，将上述 3 页从 Misc Technique 索引移动到 Family 索引。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-misc-jail-format-family-retune-20260612-163714.zip`。

## 2026-06-12 — Web 注入/解析器/反序列化 family 化

- 继续执行 technique 优先的语义健康检查，处理 Web 中仍含大量模板化变体表、且跨多个 sink 或解析器语义的页面。
- 将 3 个原 `type: technique` 页面改为 `type: family`：
  - `xml-command-and-graphql-injection.md`：XXE/XML、命令注入、GraphQL、PHP 变量变量、可预测文件名和正则替换绕过分流。
  - `parser-wrapper-and-legacy-ssrf-tricks.md`：解析器差异、PHP wrapper、URL parser/curl 差异、PDF/LaTeX 内部资源加载、SSRF 到内部协议和 legacy SSRF 分流。
  - `php-java-python-deserialization.md`：Java/Python/PHP/.NET 反序列化、gadget、包装层、SECRET_KEY/cookie 和 parser 内部 pickle 加载分流。
- 未并入 Web 首轮页：这三页都承担首轮之后的二级判断，直接合并会让 `web-first-pass-triage-and-chain-patterns.md` 过宽。
- 暂不拆出 `xxe.md`、`command-injection.md`、`graphql-injection.md`、`java-deserialization.md` 等小页：当前 raw 多为短案例集合，先保留 family 路由；后续积累足够 raw/WP 后再拆具体 technique。
- 同步更新 `index.md`，将上述 3 页从 Web Technique 索引移动到 Family 索引。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-web-injection-parser-deser-family-retune-20260612-164046.zip`。

## 2026-06-12 — Crypto 格/代数/证明系统 family 化

- 继续执行 technique 优先的语义健康检查，处理 Crypto 中明显承担算法族分流职责的页面。
- 将 4 个原 `type: technique` 页面改为 `type: family`：
  - `lattice-and-lwe.md`：HNP/EHNP、截断 LCG、LWE/Ring-LWE、CVP/SVP、orthogonal lattice 和 subset sum 分流。
  - `number-theory-and-algebra-attacks.md`：DLP、Coppersmith、approximate GCD、p-adic/Hensel、GF(2) 线性代数和特殊代数结构分流。
  - `homomorphic-and-exotic-algebra.md`：Paillier/Goldwasser-Micali/ElGamal 同态 oracle、braid/tropical/矩阵群、FPE、BB84 和差分隐私分流。
  - `zkp-secret-sharing-and-proof-systems.md`：ZKP、garbled circuit、Shamir、Groth16、DV-SNARG、KZG/pairing 和阈值签名结构分流。
- 未并入 `crypto-parameter-triage-family.md`：总入口负责首轮参数分流，这四页负责进入格、数论、低频代数和证明系统后的二级判断。
- 暂不拆 HNP、LWE、Paillier、Groth16、Shamir 等小页：当前 raw 多为算法族集合，先保留 family 路由；后续单一路线积累足够 raw/WP 后再拆具体 technique。
- 同步更新 `index.md`，将上述 4 页从 Crypto Technique 索引移动到 Family 索引。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-crypto-algebra-family-retune-20260612-164434.zip`。

## 2026-06-12 — Forensics 图像/存储/网络证据源 family 化

- 继续执行 technique 优先的语义健康检查，处理 Forensics 中按证据源组织、但仍标为 technique 的页面。
- 将 4 个原 `type: technique` 页面改为 `type: family`：
  - `image-bitplane-qr-and-jpeg-stego.md`：JPEG/PNG/BMP/GIF、bitplane、QR tile、帧差、DCT/slack/thumbnail 和图像重组分流。
  - `disk-memory-vm-and-container-forensics.md`：磁盘、内存、VM snapshot、core dump、Android、Docker layer、云存储和勒索脚本残留的载体入口分流。
  - `filesystems-memory-dumps-and-raid.md`：分区/文件系统、ZFS/APFS/HFS+、minidump、VMDK sparse、RAID、TrueCrypt/VeraCrypt 和数据库历史恢复分流。
  - `network-covert-auth-and-reassembly.md`：PCAP covert channel、NTLMv2/MS-SNTP/RADIUS/RDP、dnscat2、RTP 音频和多层流量重组分流。
- 未合并 `disk-memory-vm-and-container-forensics.md` 与 `filesystems-memory-dumps-and-raid.md`：前者负责载体和流程选择，后者负责底层恢复模式；直接合并会让页面过宽。
- 未并入 `cross-domain-forensics-technique-map.md`：总 map 负责跨证据源下一跳，这四页负责进入图像、网络、磁盘/内存后继续二级判断。
- 同步更新 `index.md`，将上述 4 页从 Forensics Technique 索引移动到 Family 索引。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-forensics-media-storage-network-family-retune-20260612-164852.zip`。

## 2026-06-12 — Pwn 核心利用页重写与分层调整

- 继续执行 technique 优先的语义健康检查，处理 Pwn 中模板化程度高的核心利用页面。
- 保留 `format-string.md` 为 `type: technique`：格式串漏洞有明确识别信号、最小证据和 exploitation primitive，适合作为具体技术页；本轮重写为 offset、leak、write、落点和过滤器绕过的可复用骨架。
- 将 2 个原 `type: technique` 页面改为 `type: family`：
  - `ret2csu-dynelf-and-shellcode.md`：ret2libc、raw syscall ROP、ret2csu、DynELF、bad-character ROP、exotic gadgets 和 shellcode stager 的后期落地分流。
  - `heap-houses-unlink-and-tcache.md`：House 系列、unlink、tcache、largebin、custom allocator、musl meta 和 allocator metadata 的 heap 路线分流。
- 未合并 `heap-houses-unlink-and-tcache.md` 与 `heap-uaf-tcache-and-custom-allocator.md`：前者偏 metadata/bin/house 路线，后者偏 UAF/对象生命周期和 primitive 形成。
- 同步更新 `index.md`，将上述 2 页从 Pwn Technique 索引移动到 Family 索引，`format-string.md` 保留在 Pwn Technique 索引。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-pwn-core-technique-family-retune-20260612-165244.zip`。

## 2026-06-12 — Crypto PRNG/stream/ECC 页面 family 化

- 继续执行 technique 优先的语义健康检查，处理 Crypto 中仍含模板化变体表、且实际承担二级分流职责的页面。
- 将 4 个原 `type: technique` 页面改为 `type: family`：
  - `mt-lcg-and-seed-recovery.md`：MT19937、`random.random()`、时间种子、C `rand()`、LCG、V8 `Math.random()` 和弱 PRNG 状态恢复分流。
  - `prng-z3-lcg-and-timing-attacks.md`：partial output、Z3、LCG 逆推、timing oracle、格式串改写和跨域 PRNG 状态补全分流。
  - `rc4-lfsr-and-keystream-reuse.md`：LFSR、RC4 bias、keystream reuse、known plaintext、custom stream 和 PCAP/DNS key 泄露分流。
  - `ecc-dlp-and-signature-attacks.md`：ECC/DLP、small subgroup、invalid/singular/anomalous curve、torsion side channel、nonce reuse 和 partial nonce 签名恢复分流。
- 未合并 `prng-z3-lcg-and-timing-attacks.md` 到 `mt-lcg-and-seed-recovery.md`：前者处理的是约束、计时和跨领域 primitive 补齐状态，第一证据与直接 PRNG 类型识别不同。
- 未合并 `rc4-lfsr-and-keystream-reuse.md` 到 PRNG 页：流密码问题的核心证据是 keystream、known plaintext、位序和复用关系，不是通用 seed recovery。
- 同步更新 `index.md`，将上述 4 页从 Crypto Technique 索引移动到 Family 索引。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-crypto-prng-stream-ecc-family-retune-20260612-165609.zip`。

## 2026-06-12 — Web auth/Node 与 Misc DNS/game 页面 family 化

- 继续执行 technique 优先的语义健康检查，处理 Web 与 Misc 中仍含模板化变体表、且实际承担二级分流职责的页面。
- 将 6 个原 `type: technique` 页面改为 `type: family`：
  - `node-and-prototype.md`：Node 原型污染、gadget、VM/sandbox escape、Happy-DOM、RSC 和内部 require/SSRF 转向分流。
  - `oauth-saml-cors-and-cicd.md`：OAuth/OIDC、SAML、CORS、Git 历史、CI/CD 变量、身份平台 API 和 TeamCity/Guacamole 链分流。
  - `auth-jwt.md`：JWT/JWE、签名 cookie、算法混淆、JWK/JKU/KID、弱 secret 和 token parser 差异分流。
  - `polyglot-url-tricks-and-ssrf-leaks.md`：URL parser 差异、gopher/CRLF、polyglot 上传、SSRF 凭据泄露和 DNS rebinding 转向分流。
  - `dns.md`：ECS、DNSSEC、zone transfer、rebinding、DNS tunnel、SPF/TXT 和解析链分流。
  - `game-state-websocket-and-wasm.md`：游戏状态、WebSocket、Flask session、WASM linear memory、de Bruijn 和资源包状态复现分流。
- 未合并 `auth-jwt.md` 到 `auth-bypass-cookies-and-hidden-routes.md`：认证总入口负责判断是否进入 token 路线，本页负责 token 内部算法、key lookup、claim 和解析差异。
- 未合并 `dns.md` 到 OSINT `web-and-dns.md`：后者是公开情报查询入口，本页处理协议行为、隧道、rebinding 和主动解析差异。
- 未合并 `polyglot-url-tricks-and-ssrf-leaks.md` 到 `parser-wrapper-and-legacy-ssrf-tricks.md`：前者聚焦 payload 自身的 URL/协议多解释，后者覆盖更宽的二段解析器。
- 同步更新 `index.md`，将上述 6 页从 Technique 索引移动到 Family 索引。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-web-auth-node-misc-family-retune-20260612-170226.zip`。

## 2026-06-12 — Pwn sandbox/kernel/heap 页面 family 化

- 继续执行 technique 优先的语义健康检查，处理 Pwn 中仍含模板化变体表、且承担二级分流职责的页面。
- 将 4 个原 `type: technique` 页面改为 `type: family`：
  - `python-vm-and-proc-sandbox-escape.md`：Python/VM、`/proc`、FUSE/CUSE、fifo、emulator、restricted shell 和 sandbox primitive 分流。
  - `kernel-uaf-race-and-slab-techniques.md`：Kernel UAF、race、SLUB/slab、freelist hardening、BPF verifier/runtime 差异和 PTE/page overlap 分流。
  - `heap-uaf-tcache-and-custom-allocator.md`：UAF、double free、tcache poisoning、custom allocator、对象生命周期和函数指针/虚表落点分流。
  - `heap-fsop-file-structure-attacks.md`：glibc FILE、stdout/stdin、FSOP、IO buffer、vtable validation 和 FILE 字段落点分流。
- 未合并 `heap-uaf-tcache-and-custom-allocator.md`、`heap-houses-unlink-and-tcache.md` 与 `heap-fsop-file-structure-attacks.md`：三者分别负责 primitive 形成、metadata/bin 技巧和 FILE 结构落点。
- 未合并 `python-vm-and-proc-sandbox-escape.md` 到 `pyjails.md`：前者偏 OS/VM/pwn primitive，后者偏 Python 语言对象链。
- 同步更新 `index.md`，将上述 4 页从 Pwn Technique 索引移动到 Family 索引。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-pwn-sandbox-kernel-heap-family-retune-20260612-171026.zip`。

## 2026-06-12 — Reverse/Forensics 硬件信号与运行时 trace 页面 family 化

- 继续执行 technique 优先的语义健康检查，处理 Reverse/Forensics 中仍含模板化变体表、且承担二级分流职责的页面。
- 将 4 个原 `type: technique` 页面改为 `type: family`：
  - `signals-and-hardware.md`：VGA/HDMI/DisplayPort、UART/I2C/SPI、RF、功耗侧信道、键盘声学、LED/视频和旧硬件介质恢复分流。
  - `hardware-isa-bootloader-and-kvm.md`：低频 ISA、固件、bootloader、KVM guest、MCU、TrustZone/EL3 和协处理器分流。
  - `runtime-patching-oracles-and-tracing.md`：运行时 patch、hook、oracle、INT3/coredump、timing 和上下文数据替换分流。
  - `signal-trace-and-packed-anti-analysis.md`：信号处理器、trace 反演、父子进程 patch、ConfuserEx/dynamic module dump 和 packed anti-analysis 分流。
- 修正 `signals-and-hardware.md` 的索引方向：该页 frontmatter 与 raw 均属 forensics 信号恢复，本轮从 Reverse Technique 列表移到 Family 表的 forensics 行。
- 未合并 `runtime-patching-oracles-and-tracing.md` 与 `signal-trace-and-packed-anti-analysis.md`：前者偏主动 patch/hook/oracle，后者偏 signal/trace/dump 证据结构。
- 同步更新 `index.md`，将上述 4 页从 Technique 索引移动到 Family 索引。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-reverse-hardware-trace-signal-family-retune-20260612-171743.zip`。

## 2026-06-12 — 剩余模板页 family 化与 file triage 重写

- 继续执行 technique 优先的语义健康检查，清理剩余含机械化“复用重点”占位句的页面。
- 将 8 个原 `type: technique` 页面改为 `type: family`：
  - `audio-frequency-and-archive-stego.md`：音频频域、SSTV、DTMF、声道 LSB、音频元数据、DeepSound 和语音特征分流。
  - `video-document-and-media-stego.md`：视频帧、PDF/文档对象、媒体容器、JXL/EXIF、ANSI escape 和运行时媒体 dump 分流。
  - `peripheral-capture.md`：USB HID、鼠标/键盘、LED Morse、Bluetooth RFCOMM 和外设 report 重组分流。
  - `oracles-recurrences-captcha-polyglots.md`：比较/timeout oracle、递推、CAPTCHA、QR 结构约束和 polyglot 分流。
  - `adversarial-ml.md`：FGSM/PGD/C&W、adversarial patch、black-box 查询、poisoning 和 backdoor 分流。
  - `llm-attacks.md`：prompt injection、jailbreak、token smuggling、context manipulation 和 tool-use exploitation 分流。
  - `pe-and-dotnet.md`：PE/.NET、配置提取、DNS C2、PyInstaller/PyArmor、YARA、shellcode 和内存证据分流。
  - `exotic-secret-sharing-rabin-and-polynomials.md`：BIP39、Asmuth-Bloom、Rabin、多项式、Vandermonde、LCG 周期和长尾原语分流。
- 重写已为 `type: family` 的 `file-triage-archives-and-one-liners.md`，去掉旧模板表，明确其只承担轻量附件首检和下一跳判断。
- 未合并 audio/video/peripheral 三页：三者分别处理音频频域、媒体容器/文档、外设 report，证据形态和工具路径不同。
- 未合并 adversarial ML 与 LLM：前者依赖模型扰动/梯度/查询反馈，后者依赖上下文、提示和工具调用边界。
- 同步更新 `index.md`，将上述页面移入 Family 索引并从 Technique 索引移除。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-final-template-family-retune-20260612-172548.zip`。

## 2026-06-13 — 剩余 Technique 语义健康检查与二级 family 收敛

- 继续按 technique 优先顺序做语义健康检查：逐页判断剩余 `type: technique` 是否存在类型错位、应合并、应拆分、应重命名或删除。
- 将 5 个原 `type: technique` 页面改为 `type: family`：
  - `ml-model-inference-extraction-and-weight-analysis.md`：query extraction、model inversion、weight diff、encoder collision、membership inference 与 task-semantic inference 的 AI/ML 模型路线分流。
  - `blockchain-smart-contract-exploitation.md`：EVM、Solana/Anchor、Sui Move、链上随机、proxy/delegatecall、ABI 和 capability/resource 约束分流。
  - `filesystem-archive-recovery-and-repair.md`：文件系统删除恢复、损坏归档、ZipCrypto 已知明文、嵌套容器、加密容器和数据库页恢复分流。
  - `file-signatures-and-flag-artifact-hunting.md`：未知文件、magic/metadata、尾随数据、未引用对象和常见编码层的首轮分流。
  - `keyboard-mouse-audio-and-physical-puzzles.md`：USB HID、鼠标/键盘、音频频率、视频轨迹、LED/Morse 和物理动作记录恢复分流。
- 保留未转换的剩余 technique 页：它们都有独立识别信号、最小证据和解法骨架，虽有关键变体，但没有承担首轮/二级目录职责。
- 未拆分 `lorenz-and-book-cipher-attacks.md`：历史机械密码和 book cipher 第一操作不同，但当前 raw 支撑集中在同一低频 cipher 页，现阶段拆分会制造弱入链小页；后续若新增 Lorenz 或 book cipher raw 再拆。
- 未合并 `path-confusion-to-signed-internal-request-chain.md`、`json-duplicate-key-hmac-parser-differential.md`、`workflow-runner-internal-api-chain.md` 与 `artifact-trust-ssrf-to-node-require-rce.md`：它们可以组成 Web 漏洞链，但第一证据分别是路径规范化、JSON parser 差异、runner 内部信任和 artifact trust/RCE，保留为独立 technique 更利于下一跳定位。
- 同步更新 `index.md`，将上述 5 页从 Technique 索引移动到 Family 索引。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-technique-family-retag-20260613-184123.zip`。
