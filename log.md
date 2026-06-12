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
- 已授权修复 raw 缺失附件链接：`raw/ai-ml/SU_theifWP.md` 中 `[app.py](./app.py)` 改为普通附件名描述，避免指向不存在的同层文件。
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
