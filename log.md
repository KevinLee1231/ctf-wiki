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

## 2026-06-18 — 物理信号、RF 与受限 shell 页关联边修复

- 继续执行 family 入链和 technique 出链抽查，修复仍带有旧 misc 模板关联边的页面。
- 更新 `keyboard-mouse-audio-and-physical-puzzles.md` 的关联技巧：改为连接 `peripheral-capture.md`、`signals-and-hardware.md`、`audio-frequency-and-archive-stego.md`、`3d-printing.md`、`rf-sdr.md` 和 `video-document-and-media-stego.md`。
- 更新 `rf-sdr.md` 的关联技巧：改为连接信号硬件、音频频域、网络重组、PCAP 协议恢复、物理信号 family 和 forensics tooling。
- 更新 `source-backdoors-and-restricted-shell-tricks.md` 的关联技巧：保留 shell jail 入口，并转向交互容器/jail、pyjail、脚本混淆、Web 首轮分流和轻量文件 triage。
- 本轮不修改 raw 正文，只修正 wiki 图谱出链。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-related-links-retune-20260618-153255.zip`。

## 2026-06-18 — Forensics technique 出链语义修复

- 继续抽查剩余 technique 的关联边，修复旧通用 forensics 链接集造成的误导。
- 更新 `3d-printing.md`：移除与区块链取证、磁盘/内存取证的无关邻接，改为连接文件首检、归档恢复、物理信号、视频媒体、图像隐写和跨域取证 map。
- 更新 `blockchain-and-transaction-forensics.md`：移除 3D 打印、音频隐写和磁盘恢复等无关邻接，改为连接智能合约 exploitation family、跨域取证 map、文件首检、网络重组、crypto 参数 triage 和 forensics tooling。
- 本轮不修改 raw 正文，只修正 wiki 图谱出链。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-forensics-technique-links-retune-20260618-153619.zip`。

## 2026-06-18 — 新转 Family 页路由表表头收敛

- 继续修正 `2026-06-13` 由 technique 转为 family 的页面，使表格语义从 technique 风格的“变体/复用重点”转为 family 风格的“首轮判断/下一跳判断”。
- 更新 `ml-model-inference-extraction-and-weight-analysis.md`、`blockchain-smart-contract-exploitation.md`、`filesystem-archive-recovery-and-repair.md`、`file-signatures-and-flag-artifact-hunting.md`、`keyboard-mouse-audio-and-physical-puzzles.md`。
- 本轮不修改 raw 正文；只调整 wiki 页的分流表达和 `updated` 日期。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-new-family-table-retune-20260618-153907.zip`。

## 2026-07-06 — Family 页残留 Technique 表头收敛

- 继续检查 family 页中残留的 technique 写法，将 `变体 / 复用重点` 表头改成更适合 family 的证据、下一跳和路线判断表达。
- 更新 `cross-domain-forensics-technique-map.md`、`cross-primitive-escape-and-hybrid-exploit-map.md`、`osint-account-public-media-correlation.md`、`pwn-first-pass-red-flags-and-protections.md`。
- 本轮不修改 raw 正文；只修正 family 页的分流表述和 `updated` 日期。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-family-table-header-retune-20260706-121546.zip`。

## 2026-07-06 — Index 当前状态日期同步

- 同步 `index.md` 的 `Updated` 日期到 `2026-07-06`，反映本轮 family 表述收敛后的当前索引状态。
- 本轮未新增或删除索引入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-index-date-sync-20260706-121716.zip`。

## 2026-07-06 — SQLi Family 边界和出链收紧

- 复核 `raw/web/sqli-filter-and-oracle.md` 后，确认 `sqli-filter-and-oracle-family.md` 的核心支撑是输入面、过滤器行为、盲注/oracle、二阶 SQLi、元数据注入和 SQLi 到 SSTI/RCE。
- 重写该页适用场景，去掉旧的泛化“技巧家族页”模板句，明确它只在核心证据仍是数据库查询语义、SQL 方言差异或 SQL oracle 时使用。
- 修剪关联技巧：移除与 SQLi 主线弱相关的浏览器侧、JWT 和 artifact RCE 出链，保留 Web 首轮、XML/GraphQL、上传/反序列化/RCE、SSTI、认证边界、解析器差异、已知组件漏洞和 Web tooling。
- 本轮不修改 raw 正文。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-sqli-family-retune-20260706-122040.zip`。

## 2026-07-06 — 核心 Family 旧模板句清理

- 清理 4 个高频 family 页中残留的早期模板句，避免页面只说明“承接相邻技巧”，而没有说明实际分流边界。
- 更新 `block-mode-misuse-family.md`：明确其边界是 block mode、nonce/IV/counter、MAC/tag、padding 和 oracle；修剪 RSA/ECC/格/代数等弱相关出链。
- 更新 `oob-jit-parser-primitives-family.md`：明确其边界是 Pwn primitive 形成，不替代 Pwn 首轮页。
- 更新 `pcap-protocol-credential-recovery-family.md`：明确其边界是 PCAP / network forensics；移除 3D 打印、区块链取证等弱相关出链，补入文件首检、归档恢复、Web 首轮和 forensics tooling。
- 更新 `vm-obfuscation-transform-family.md`：明确其边界是执行模型被隐藏或改写时的 trace、lifting、patch、oracle 和约束恢复路线。
- 本轮不修改 raw 正文。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-core-family-template-retune-20260706-122225.zip`。

## 2026-07-06 — Family 页“关键变体”标题分流化

- 将仍为 `type: family` 且使用 `## 关键变体` 标题的 16 个页面改为分流语义标题，避免 family 页继续呈现 technique 风格。
- 涉及页面包括 blockchain、cross-domain forensics、cross-primitive pwn、crypto parameter triage、ECC/signature、file triage、filesystem recovery、physical signal、misc triage、ML model、PRNG、OSINT account、constraint/timing PRNG、Pwn first-pass、stream cipher 和 SQLi family。
- 本轮只改章节标题，不改 raw 正文。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-family-key-variant-heading-retune-20260706-122417.zip`。

## 2026-07-06 — 泛化 Tags 元数据补强

- 继续检查查询入口质量，处理 tags 只有“方向 + type”的页面，避免页面只能靠文件名被发现。
- 为 5 个核心 family 页补充具体主题 tags：block mode/MAC/oracle、OOB/JIT/parser primitive、PCAP/network credential、SQLi/oracle/filter bypass、VM/obfuscation/SMC/tracing。
- 为 8 个 technique 页补充具体主题 tags：3D printing/G-code、auth edge cases、blockchain transaction forensics、emulator/float/hash primitive、Lorenz/book cipher、race/TOCTOU、RF/SDR/IQ、source backdoor/restricted shell。
- 将 `rf-sdr.md` 的 `skills` 同步为 `[ctf-forensics, ctf-misc]`，与其物理信号/取证定位和相邻页面一致。
- 本轮不修改 raw 正文。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-generic-tags-retune-20260706-122814.zip`。

## 2026-07-06 — 跨方向 Skills 元数据收敛

- 继续执行 semantic health check 的元数据层检查，判断 `tags:`、`skills:`、raw 来源方向和实际入链是否一致。
- 更新 `audio-frequency-and-archive-stego.md`：补入 `misc` tag 和 `ctf-misc`，因为音频频域/SSTV/DTMF/语音认证同时由 Forensics 和 Misc 首轮进入。
- 更新 `blockchain-and-transaction-forensics.md`：补入 `ctf-osint`，因为公开链浏览器、地址标签、交易备注和 public metadata 具有 OSINT 入口价值；仍保留为 Forensics technique。
- 更新 `pyjails.md`：补入 `web` tag 和 `ctf-web`，因为 Web 表达式沙箱和 Flask/eval 包装会从 Web 首轮直接转入本页。
- 更新 `race-condition-and-concurrency-exploits.md`：从单一 Pwn technique 调整为 Pwn/Web/Pentest/Misc 共享 technique，并在 `index.md` 中移到 Cross-Direction technique 分组。
- 本轮不修改 raw 正文；只修正 wiki 元数据和索引入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-skill-metadata-retune-20260706-123342.zip`。

## 2026-07-06 — 跨方向入口元数据二次收敛

- 继续阅读剩余候选页，区分“raw 位于 misc 但语义已被专项吸收”和“实际需要跨 skill 进入”的页面。
- 更新 `mt-lcg-and-seed-recovery.md`：补入 `web` tag 和 `ctf-web`，因为页面覆盖 Web token/key/鉴权随机数，且 Web 首轮页直接把 Flask/MT19937 案例转入本页。
- 更新 `vm-z3-sandbox-and-game-basics.md`：补入 `pwn` tag 和 `ctf-pwn`，因为自定义 VM opcode、SP 越界和解释器 primitive 已由 Pwn 首轮页直接引用。
- 更新 `web-and-dns.md`：将 tag `web` 改成 `public-web`，明确它是 OSINT 公开来源入口，不是 Web 漏洞利用入口，因此不补 `ctf-web`。
- 同步调整 `index.md` 的方向列和摘要，使 family 入口的跨方向属性更准确。
- 本轮不修改 raw 正文。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-cross-entry-metadata-retune-20260706-123651.zip`。

## 2026-07-06 — 剩余机械元数据候选保留说明

- 复跑 `tags/skills/rawDirs` 机械扫描后，剩余候选主要来自 raw 文件物理目录为 `misc/`，但页面语义已稳定归入更具体专项；这些不应为了消除扫描项而机械补 `ctf-misc`。
- 暂保留 `filesystem-archive-recovery-and-repair.md`、`linux-git-browser-and-container-forensics.md`、`network-covert-auth-and-reassembly.md` 为 `ctf-forensics`：它们的核心证据是文件系统、Git/Docker/browser、PCAP/协议重组，而不是 Misc 后备题型。
- 暂保留 `known-cves-and-n-day-exploits.md` 为 `ctf-web`：raw 中的 misc 来源只是赛题分类，技术边界仍是 Web/N-day/组件版本判断。
- 暂保留 `llm-attacks.md` 为 `ctf-ai-ml`：LLM prompt/tool-use 是 AI-ML 专项入口，不因 raw 来源在 misc 而降级成 Misc。
- 暂保留 `malware-c2-session-key-and-protocol-recovery.md` 和 `scripts-and-obfuscation.md` 为 Malware 主入口：`.eml`、脚本 stage、C2/session key 和载荷还原应从 Malware/Forensics 进入。
- 暂保留 `osint-account-public-media-correlation.md` 为 `ctf-osint`：公开账号、媒体、archive 和身份链属于 OSINT，不因 WP 放在 misc 而补 `ctf-misc`。
- 本轮不修改 raw 正文；本节用于避免后续把 raw 物理目录误当作页面 skill 归属。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-residual-metadata-audit-note-20260706-123831.zip`。

## 2026-07-06 — 低入链 Technique 出链复核

- 继续审计剩余 technique 页的入链和出链，优先检查低入链页面是否仍有独立识别信号、是否需要并回 family，以及关联边是否仍是旧泛化模板。
- 复核 `lorenz-and-book-cipher-attacks.md`：继续保留为 technique，不拆分 Lorenz/book cipher；当前 raw 支撑集中且上游 `classical-xor-and-substitution-ciphers.md` 已明确承担 family 分流。
- 修正该页关联技巧：移除 block mode、ECC、嵌入式 Python 等弱相关邻接，改为连接古典/XOR family、stream cipher/keystream、轻量编码、异常格式和 crypto 参数分流。
- 本轮不修改 raw 正文。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-lorenz-related-links-retune-20260706-124114.zip`。

## 2026-07-06 — Cross-Direction Technique 标签与出链收敛

- 继续检查 technique 出链方向一致性，处理 `skills:` 已经跨方向但 `tags:` 或索引仍表现成单方向的页面。
- 更新 `bgp-rpki-route-hijack.md`：补入 `pentest` tag，并从 Misc technique 分组移动到 Cross-Direction technique 分组；该页同时服务 Misc 网络题和 Pentest/内网路由 pivot。
- 更新 `protocol-relay-and-internal-service-injection.md` 与 `workflow-runner-internal-api-chain.md`：补入 `pentest` tag，保留在 Web technique 分组，因为首轮入口仍是 Web 应用/内部 API。
- 更新 `vmp-client-server-smc-rc4-recovery.md`：将弱相关的游戏状态出链替换为 `compare-breakpoint-plaintext-recovery.md`，更贴近动态比较点、明文 buffer 和协议恢复主线。
- 本轮不修改 raw 正文。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-cross-technique-tags-and-links-20260706-124257.zip`。

## 2026-07-06 — Family 页 Technique 模板标题收敛

- 继续检查 `type: family` 页面是否仍保留 technique 风格标题。
- 将 20 个 family 页中的 `## 适用场景` 改为 `## 作用边界`，`## 解法骨架` 改为 `## 分流流程`，使页面形态更贴近 family 的首轮判断、下一跳和 pivot 职责。
- 涉及页面包括 block mode、smart contract、cross-domain forensics、cross-primitive pwn、crypto parameter triage、ECC/signature、file triage、filesystem recovery、physical signal、misc triage、ML model、PRNG、OOB/JIT/parser、OSINT account、PCAP、Pwn first-pass、stream cipher、SQLi 和 reverse VM family。
- 本轮只调整标题语义，不修改 raw 正文，不改变页面引用关系。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-family-heading-retune-20260706-124734.zip`。

## 2026-07-06 — 方向级 Tooling 页失败转向补强

- 继续修复 tooling 页“只有工具清单、缺少失败转向”的问题。
- 更新 `osint-tooling.md`：补充搜索漂移、媒体线索、需要登录/漏洞利用、用户名枚举命中过多时的转向。
- 更新 `ai-ml-tooling.md`：补充 HTTP/API 误判、无概率反馈、模型文件加载失败和重量级依赖安装边界。
- 更新 `malware-tooling.md`：补充静态 IOC 失败、PCAP-only、动态执行风险和 Office/PowerShell/.NET 反编译矛盾时的转向。
- 本轮不修改 raw 正文，不新增工具安装步骤。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-direction-tooling-failure-routes-20260706-124957.zip`。

## 2026-07-06 — 核心方向 Tooling 页失败转向补强

- 继续处理高频方向工具页的“工具清单化”风险。
- 更新 `web-tooling.md`：补充 Burp/curl baseline 不足、目录扫描误报、自动扫描无结果、进入内网/runner/token 阶段后的转向。
- 更新 `pwn-tooling.md`：补充 crash 归类不足、本地/远程差异、gadget 不可用、kernel/QEMU 不稳时的转向。
- 更新 `crypto-tooling.md`：补充基础分解失败、Sage/LLL/Z3 无解、RsaCtfTool 不覆盖、跨 Web/PCAP 协议证据时的转向。
- 本轮不修改 raw 正文，不新增工具安装步骤。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-core-tooling-failure-routes-20260706-125102.zip`。

## 2026-07-06 — 剩余方向 Tooling 页失败转向补强

- 完成剩余方向级 tooling 页的失败转向补齐。
- 更新 `reverse-tooling.md`：补充反编译不可读、反调试、最终比较点可断、完整环境模拟等转向。
- 更新 `misc-tooling.md`：补充误入 Misc、编码/QR 无结果、jail 部分输出、Z3/SAT 无解时的转向。
- 更新 `pentest-tooling.md`：补充扫描面过大、认证/权限错误、隧道不通、发现主障碍属于 Web/Misc/Pwn 时的转向。
- 更新 `forensics-tooling.md`：补充首检矛盾、PCAP 导出物、磁盘/内存输出过多、隐写工具无命中时的转向。
- 本轮不修改 raw 正文，不新增工具安装步骤。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-remaining-tooling-failure-routes-20260706-125212.zip`。

## 2026-07-06 — Web 首轮自引用案例出链收敛

- 继续语义健康检查，针对 `web-first-pass-triage-and-chain-patterns.md` 中仍回跳到自身的案例行逐篇读取对应 raw，确认这些行已有更具体的可复用模式。
- 将 13 条 Web 自引用案例改到具体下一跳：PHP 反序列化/LFI、XSS/JSONP、前端业务状态信任、Vite CVE+CL/TE 走私、更新包信任链、uid 映射+原型污染、pyjail、CMS 上传权限绕过、Koishi 路径穿越+配置表达式、Hessian gadget 和前端混合加密登录爆破。
- 本轮不修改 raw 正文；只调整 family 页案例表的复用信号和下一跳，避免首轮 family 自身成为案例归宿。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-web-selflink-case-reroute-20260706-125826.zip`。

## 2026-07-06 — Reverse 首轮自引用案例出链收敛

- 继续语义健康检查，复核 `reverse-first-pass-workflow-and-debugging.md` 中仍回跳到自身的案例行。
- 将 16 条 Reverse 自引用案例改到更具体入口：Excel/RISC-V VM、Heaven's Gate/WoW64、最终比较/常量表反推、非标准 Base64、汇编文本模拟、假 exe 文件首检、VMP+RC4-like 文件格式、游戏协议整数下溢、Godot PCK/DLL/SMC、直接运行验证、Unity/IL2CPP+DP、固件容器/ELF 修复、私有协议 block 变换、Windows 驱动协同校验、APK native 加密签名和 `setjmp/longjmp` 控制流。
- 暂不新建“reverse 协议恢复”页：`SU_protocolWP`、`SU_old_binWP`、`NCTF2026-鸡爪流高手-wp` 等案例有共同点，但现阶段可由 loader/固件、runtime tracing、game-state 和 block-mode family 承载；若后续出现更多同类 raw，再拆为独立 family。
- 本轮不修改 raw 正文；只调整首轮 family 的案例出链。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-reverse-selflink-case-reroute-20260706-130313.zip`。

## 2026-07-06 — 剩余自引用案例和 AI-ML Technique 拆分

- 处理剩余 5 条自引用案例：`SU_forensicsWP` 改到 Windows artifact / 磁盘取证入口，`LilacCTF2026-sky-is-ours-wp` 改到 OSINT tooling，避免 family 案例行继续指向自身。
- 新建 `linear-model-parameter-recovery.md` 作为 AI-ML technique，承载 `SU_谁是小偷WP`、`SU_我不是神偷WP`、`SU_theifWP` 中共同的无激活线性模型参数恢复、basis query、state_dict 形状泄露和上传校验缺口利用。
- 更新 `ml-model-inference-extraction-and-weight-analysis.md`：AI-ML family 继续负责模型攻击分流，具体的线性参数恢复解法转入新 technique。
- 更新 `index.md` 增加 AI / ML technique 入口。
- 本轮不修改 raw 正文；新增页和 family 案例只引用 raw 作为证据。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-final-selflink-and-ai-technique-20260706-130548.zip`。

## 2026-07-06 — Auth Edge Cases 改为 Family

- 继续 technique 语义检查，确认 `auth-edge-cases-and-protocol-bypasses.md` 同时覆盖 hash bucket、Unicode normalization、SRP/DH 参数校验和 AQL/NoSQL 权限提升，不是单一 technique。
- 将该页 frontmatter 从 `type: technique` 改为 `type: family`，并把章节标题调整为 family 语义：作用边界、分流流程、认证边界分流、合并与拆分结论。
- 更新 `index.md`：从 Web technique 列表移除该页，加入 Web family 索引。
- 本轮不修改 raw 正文。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-auth-edge-family-retune-20260706-131011.zip`。

## 2026-07-06 — Reverse Compare 泛化案例收敛

- 继续查找高频重复案例句，发现 `reverse-first-pass-workflow-and-debugging.md` 中 15 条 raw 都被概括成“复杂校验最终落到比较点”。
- 逐篇复核对应 raw 后重写这些案例行：将 Go/LoongArch/SM4、`atexit`+RC4、DWORD 约束+3DES、明文 strings、Z3 哈希、Base64、TEA/XTEA、Go RSA、ptrace 自解密+RC4、线性方程、UPX、VM、AES-ECB 和多层驱动 IOCTL 分别转到更准确的 family/technique/tooling。
- 这轮减少了 `compare-breakpoint-plaintext-recovery.md` 的误导性入链；后续需要继续复核该 technique 自身的 raw 支撑是否仍准确。
- 本轮不修改 raw 正文。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-reverse-generic-compare-rows-20260706-131309.zip`。

## 2026-07-06 — Compare Technique Raw 支撑收紧

- 继续复核 `compare-breakpoint-plaintext-recovery.md` 的 raw 支撑，确认 `ACTF2026-flagchecker-wp` 和 `SU_LockWP` 更准确的入口分别是 Go/ISA/shellcode 和 loader/Windows driver IOCTL，不应继续作为 compare-breakpoint 代表案例。
- 从该 technique 的案例表中移除上述两条，仅保留 `Spirit2026-5-xxxtea-wp` 作为“复杂算法外观下运行到比较点直接 dump 明文”的直接证据。
- 将 `reverse-first-pass-workflow-and-debugging.md` 中 `Bugku-GoldDigger-wp` 的下一跳从 compare-breakpoint 改为 `self-decrypting-strings-and-lattice-patterns.md`，因为它是静态置换表和目标数组反填，不是动态比较点明文恢复。
- 本轮不修改 raw 正文。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-compare-page-raw-retune-20260706-131453.zip`。

## 2026-07-06 — Web 文件链泛化案例收敛

- 继续查找高频重复案例句，发现 `web-first-pass-triage-and-chain-patterns.md` 中 9 条 raw 都被概括成“文件、路径、模板、反序列化或上传链”。
- 逐篇复核对应 raw 后重写这些案例行：将 go-drive/open_basedir/StorageBox TOCTOU、Java cookie 反序列化、Rust Serde 状态绕过、Bun shell raw 命令注入、SQLite join 字段覆盖、CMS 反序列化到 SQLi 到模板执行、DNS rebinding 到 Docker API、JEECG ZIP Slip 写 JSP、Pandoc docx 资源嵌入分别转到更准确下一跳。
- 本轮不修改 raw 正文；只细化 Web 首轮 family 的 raw 案例概括。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-web-generic-file-chain-rows-20260706-131651.zip`。

## 2026-07-06 — Source Backdoors 改为 Family

- 继续 technique 语义健康检查，确认 `source-backdoors-and-restricted-shell-tricks.md` 同时覆盖源码隐藏后门、受限 shell、HISTFILE/`bash -v`、rvim 插件逃逸和工具副作用，不是单一 technique。
- 将该页 frontmatter 从 `type: technique` 改为 `type: family`，并把正文改成作用边界、分流流程和合并/拆分结论。
- 更新 `index.md`：从 Misc technique 列表移除该页，加入 Misc family 索引。
- 本轮不修改 raw 正文。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-source-backdoor-family-retune-20260706-continue.zip`。

## 2026-07-06 — Packers / Deobfuscation 改为 Family

- 继续 technique 语义健康检查，确认 `packers-deobfuscation-and-debug-automation.md` 同时承担壳、商业虚拟化、控制流混淆、动态解密、patch 和 trace 自动化的路线分流，不是单一 technique。
- 将该页 frontmatter 从 `type: technique` 改为 `type: family`，并删除原先“保留为 technique”的自解释，改成受保护逆向题的降维分流边界。
- 更新 `index.md`：从 Reverse technique 列表移除该页，加入 Reverse family 索引。
- 本轮不修改 raw 正文。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-packers-family-retune-20260706-continue.zip`。

## 2026-07-06 — Crypto 首轮泛化案例收敛

- 继续检查 `crypto-parameter-triage-family.md` 的 raw 案例概括，针对 8 条仍使用“数论/代数方程”或“RSA 参数异常”的泛化行逐篇读取 raw。
- 将 CKKS 排名改到同态 family，将 bit 级 XOR key schedule 改到 keystream family，将小系数 RSA hint 改到 lattice family；其余行补充具体信号：十进制前缀模 1 区间、二次型判别式和小根、Pisano period、小模数 `m>n` 字符约束格、指数 bit-flip 与低字节 oracle。
- 同步修正首轮分流表的 WP 数，新增 `rc4-lfsr-and-keystream-reuse.md` 作为 Crypto 首轮下一跳。
- 本轮不修改 raw 正文。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-crypto-family-specific-case-retune-20260706-continue.zip`。

## 2026-07-06 — Pwn 首轮泛化案例收敛

- 继续检查 `pwn-first-pass-red-flags-and-protections.md` 的 raw 案例概括，针对 10 条仍使用“堆生命周期”或“栈溢出/基础覆盖”的泛化行逐篇读取 raw。
- 将 off-by-null + house of obstack、Rust GC UAF/vtable、自定义 allocator unsafe unlink、largebin 改 `mp_.tcache_bins`、Hexagon ROP、格式串 `_exit` 控制流、PCI/MMIO OOB、V8/J2V8 n-day、kernel page-cache 写入和 libevent callback ORW 分别改到更准确下一跳。
- 本轮不修改 raw 正文；只修正 Pwn family 案例表的复用信号和路由。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-pwn-family-specific-case-retune-20260706-continue.zip`。

## 2026-07-06 — Web SSRF/Auth 桶化案例收敛

- 继续检查 `web-first-pass-triage-and-chain-patterns.md` 的重复“SSRF/内部服务/可信 artifact 链”案例行，逐篇读取 HGAME/RCTF raw，并复用已有 Spirit technique 页的拆分边界。
- 将 `my-little-assistant` 改为 MCP/Playwright 到 localhost JSON-RPC 工具调用，`my-monitor` 改为 `sync.Pool` 残留字段污染管理员命令，`rootkb-dash` 改为 Redis/Celery pickle，`rootkb` 改为可写 `LD_PRELOAD` 共享库。
- 将 Spirit 三个 Web 案例分别路由到 artifact trust、duplicate-key HMAC + workflow runner、path confusion signed internal request，避免全部落到 artifact trust 页。
- 本轮不修改 raw 正文。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-web-ssrf-auth-case-retune-20260706-continue.zip`。

## 2026-07-06 — Web Auth 桶化案例收敛

- 继续检查 `web-first-pass-triage-and-chain-patterns.md` 的“认证、会话或 token 语义异常”重复案例行，逐篇读取 ACTF/RCTF/SU raw。
- 将 `12307` 拆成 SSO continuation 信任、ORDER BY 盲注和 duplicate-key print plan；将 `aaa26` 拆成 Mongo 正则盲注、JWT secret 泄露和 ImageMagick `text:/flag` 渲染。
- 将 `RCTF2025-auth` 改为类型转换 + SAML XML Signature Wrapping；将 author 系列改为 meta/CSP/XSS/popover；将 `SU_Note` 系列分别改为 Set-Cookie 透传和内网 bot XSS。
- 本轮不修改 raw 正文。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-web-auth-bucket-retune-20260706-goal.zip`。

## 2026-07-06 — Reverse VM 桶化案例收敛

- 继续检查 `reverse-first-pass-workflow-and-debugging.md` 中重复的“VM/解释器/混淆变换”案例行，逐篇读取 7 篇 Reverse raw。
- 将 `ACTF2026-virtualnpu` 改为 CUDA fatbin/NPU bytecode 与 RC4/S-box 变换；将 `Bugku-EasyVT` 改为 VT-x VM-exit 调度壳下的 TEA/RC4；将 `HGAME2026-marionette` 改为 `ptrace`/`int3` trace 后恢复 AES-NI。
- 将 `NCTF2026-vm-encryptor` 和 `RCTF2025-onion` 保留在 VM 路线，但补充 opcode disassembly、魔改 Base64、PC/HIPC/LOTAG/HITAG 和自动逆算信号。
- 将 `SU_West` 从泛化 VM 改为 permutation dispatch 状态机 + Unicorn 推进；将 `VNCTF2026-delicious-obf-ez-maze` 改为控制流混淆/SMC/反调试与魔改 UPX/MFC 迷宫路线。
- 本轮不修改 raw 正文；只修正 Reverse 首轮 family 的 raw 案例概括和下一跳。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-reverse-vm-bucket-retune-20260706-goal.zip`。

## 2026-07-06 — Index 占位链接校验修复

- 将 `index.md` 查询流程中的示例占位路径改为非链接文字，避免结构校验把 `wiki/<family|technique|tooling>.md` 误判成缺失页面。
- 本轮不改变任何真实 wiki 页面入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-index-placeholder-link-fix-20260706-goal.zip`。

## 2026-07-06 — 低入链 Technique 表述收紧

- 复核 4 个低入链 technique 页及其 raw 支撑：`blockchain-and-transaction-forensics.md`、`path-confusion-to-signed-internal-request-chain.md`、`vmp-client-server-smc-rc4-recovery.md`、`3d-printing.md`。
- 暂不合并或删除：四者的首轮证据分别是链上交易图、路径规范化差异、VMP/SMC client-server 协议恢复和 3D/CAD/G-code 载体，识别信号互不相同。
- 将这 4 页的“关键变体 / 变体 / 复用重点”模板表头改成具体证据分支表述，减少 technique 页的泛化模板感。
- 本轮不修改 raw 正文，不改变页面类型和索引入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-low-inlink-technique-table-retune-20260706-goal.zip`。

## 2026-07-06 — Family 与 Web Technique 分支表述收紧

- 继续处理剩余 `变体 / 复用重点` 表头，逐页读取 `auth-edge-cases-and-protocol-bypasses.md`、`source-backdoors-and-restricted-shell-tricks.md`、`csp-xsleak-and-browser-exfiltration.md` 和 `json-duplicate-key-hmac-parser-differential.md`。
- 保留前两者为 family：它们负责身份边界和隐藏入口分流；保留后两者为 technique：它们分别围绕浏览器外带 oracle 和 JSON parser/HMAC 差异。
- 将表头改为身份边界、隐藏入口、浏览器外带和解析差异语义，不改变页面类型、入链或 raw 引用。
- 本轮不修改 raw 正文。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-family-and-web-technique-table-retune-20260706-goal.zip`。

## 2026-07-06 — Data Interpretation Technique 重命名

- 复核 `emulator-float-and-hash-exploits.md` 及其 raw，确认共同抽象是 data interpretation 导致宿主内存 primitive，而不是 emulator/float/hash 三个题名的简单并列。
- 将页面重命名为 `data-interpretation-memory-primitives.md`，并把正文标题与分支表述改为“数据解释分支”。
- 手工更新 `index.md`、`oob-jit-parser-primitives-family.md`、`cross-primitive-escape-and-hybrid-exploit-map.md`、`race-condition-and-concurrency-exploits.md` 和 `pwn-first-pass-red-flags-and-protections.md` 的入链。
- 本轮不修改 raw 正文；raw 中旧文件名只作为历史文本保留。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-data-interpretation-technique-rename-20260706-goal.zip`。

## 2026-07-06 — 剩余 Technique 分支表述收紧

- 逐篇读取剩余 6 个带 `变体 / 复用重点` 表头的 technique 页：`format-string.md`、`lorenz-and-book-cipher-attacks.md`、`malware-c2-session-key-and-protocol-recovery.md`、`protocol-relay-and-internal-service-injection.md`、`race-condition-and-concurrency-exploits.md`、`rf-sdr.md`。
- 保留它们为 technique：格式串、历史/书本密码、C2 会话密钥、relay 内网协议注入、并发窗口和 RF/IQ 解调的最小证据与首轮动作互不相同。
- 将表头改为具体场景、结构、证据形态、注入条件、竞争形态和信号形态，消除剩余 technique 模板句。
- 本轮不修改 raw 正文，不改变页面类型或索引入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-remaining-technique-table-retune-20260706-goal.zip`。

## 2026-07-06 — Packers Family 标题收紧

- 复查 family 页标题残留，发现 `packers-deobfuscation-and-debug-automation.md` 仍使用 technique 风格的 `适用场景` 和 `关键变体` 标题。
- 将其改为 `作用边界` 与 `保护类型分流`，并把表头改为 `保护形态 / 判断与路线`，与该页“受保护逆向题分流入口”的定位一致。
- 本轮不修改 raw 正文，不改变页面类型或索引入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-packers-family-heading-retune-20260706-goal.zip`。

## 2026-07-06 — Reverse 剩余泛化案例收敛

- 继续检查 `reverse-first-pass-workflow-and-debugging.md` 的重复案例描述，逐篇读取 13 篇 raw，确认它们虽然仍属于 Reverse 首轮 family，但首轮信号并不相同。
- 将移动端/平台泛化行改成魔改 Lua VM、iOS 手势加权和、Electron JS/WASM license；将语言运行时泛化行改成 Native AOT Twofish-like、Flutter/Dart+Hermes+native、Electron native `.node` VM。
- 将 Python/Cython 泛化行改成 PyInstaller runtime hook、lambda calculus 到 GF(2^8) 插值、Cython `.pyd` AES ShiftRows 变体；将壳/trace 泛化行改成信号 handler RC4、mshta/HTA/PowerShell、VMP client-server SMC/RC4、内核驱动 PTE hook。
- 本轮不修改 raw 正文，只修正 Reverse 首轮 family 的 raw 案例概括和下一跳。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-reverse-remaining-generic-buckets-20260706-goal.zip`。

## 2026-07-06 — Web SQL 桶化案例收敛

- 继续检查 `web-first-pass-triage-and-chain-patterns.md` 中 5 条重复 SQL/数据库注入描述，逐篇读取对应 raw。
- 将 `gomysql` 改为 MariaDB 多语句/UDF 到 Go 自写模板 unsafe 执行链；将 `API 2: The SeQueL` 改为 PostgreSQL UNION 与枚举列占位；将 `safe-sql` 改为 PostgreSQL `E'...'` 反斜杠逃逸与 BRIN autosummarize CVE 提权。
- 将 `SU_jdbc-master` 从 SQLi 桶移出，改为 Unicode 路径绕过 + JDBC/Kingbase/Spring XML beans；将 `SU_sqli` 补充 Go WASM 前端签名与浏览器指纹前置条件。
- 本轮不修改 raw 正文，只修正 Web 首轮 family 的 raw 案例概括和下一跳。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-web-sqli-bucket-retune-20260706-goal.zip`。

## 2026-07-06 — Pwn 剩余泛化案例收敛

- 继续检查 `pwn-first-pass-red-flags-and-protections.md` 的 11 条重复泛化描述，逐篇读取 raw 后细化 primitive、保护约束和落点。
- 将 GPU/Mali one-shot write、Lua userdata UAF、ret2syscall `RAX=read 返回值`、生产者消费者 race、panic 地址 oracle、VM queue OOB、SQLite FTS4 trusted_schema 绕过、V8/WASM 类型混淆、PHP 扩展 off-by-one、Rsync CVE 链和 VM syscall ABI 分别改成具体首轮信号。
- 修正明显误路由：`steins;gate` 不再归并发 race；`Bugku-syscall` 不再归 seccomp；`trustSQL-plus` 和 `real-world` 补到已知组件/CVE；`ezphp` 补到 PHP 扩展内存破坏。
- 本轮不修改 raw 正文，只修正 Pwn 首轮 family 的 raw 案例概括和下一跳。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-pwn-remaining-generic-buckets-20260706-goal.zip`。

## 2026-07-06 — Crypto DLP/ECC 桶化案例收敛

- 继续检查 `crypto-parameter-triage-family.md` 中 8 条重复 “DLP/曲线/同源/配对/签名 nonce” 描述，逐篇读取 raw。
- 将矩阵 DLP 和商环 DLP 从 ECC 桶中拆出：`eezzdlp` 改为特征值 p-adic log，`ezdlp` 改为 determinant 降维，`nestdlp` 改为商环乘法矩阵 determinant + 汉明重量约束。
- 将真正曲线/同源/签名案例细化：`ezcurve` 改为 ECC noisy x-coordinate oracle + LLL，`repairing` 改为 pairing/ElGamal 重随机化，`SU_Isogeny` 改为 CSIDH CI-HNP/Coppersmith，`hd-is-what` 改为 LCG 线性去混淆 + Castryck-Decru，`schnorr` 改为重复 commitment special soundness。
- 本轮不修改 raw 正文，只修正 Crypto 首轮 family 的 raw 案例概括和下一跳。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-crypto-dlp-ecc-bucket-retune-20260706-goal.zip`。

## 2026-07-06 — Crypto PRNG 桶化案例收敛

- 继续检查 `crypto-parameter-triage-family.md` 中 6 条重复 PRNG/seed 描述，逐篇读取 raw 后确认它们都属于随机数恢复，但首轮证据形态不同。
- 将 `noisy-forest` 改为中文文本冗余泄露 MT bitstream；将 `rng-game` 改为 CPython 大整数 seed 扩展碰撞；将两道 `yet-another-*mt-game` 分别改为 Sage/GMP MT 线性恢复和 Python shuffle seed 逆置乱后恢复 GMP MT。
- 将 `SU_Prng` 改为 256-bit LCG 的 XOR/rotate 包装恢复；将 `numberguesser` 改为少量 hint 下的 Python MT untemper/twist 反推 64-bit seed。
- 本轮不修改 raw 正文，只修正 Crypto 首轮 family 的 raw 案例概括和下一跳。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-crypto-prng-bucket-retune-20260706-goal.zip`。

## 2026-07-06 — Crypto Block 桶误归类修正

- 继续检查 `crypto-parameter-triage-family.md` 中 5 条重复分组模式描述，逐篇读取 raw 后确认其中 3 条并不是 block mode / IV / padding / oracle 误用。
- 将 `myBlock` 从 block 桶改为 `GF(2^16)` 低度 Feistel 插值和秩下降；将 `NCTF2026-encryption` 保留在 block 路由但补充魔改 AES 线性化和 GF(2) 仿射求逆；将 `SU_AES` 改为 S-box 与 round key 状态不同步。
- 将 `SU_Restaurant` 改为 tropical semiring 矩阵构造；将 `ezov` 改为 UOV/OV 二次型结构恢复与签名伪造，并同步补充 `homomorphic-and-exotic-algebra.md` 的 UOV/OV 入口和案例索引。
- 本轮不修改 raw 正文，只修正 Crypto 首轮 family 的 raw 案例概括、下一跳和相邻 family 的识别边界。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-crypto-block-bucket-reroute-20260706-goal.zip`。

## 2026-07-06 — Crypto RSA 桶化案例收敛

- 继续检查 `crypto-parameter-triage-family.md` 中 5 条重复 RSA 参数异常描述，逐篇读取 raw 后确认它们都应保留在 RSA family，但首轮信号不同。
- 将 `myRSA` 改为 CM 曲线/ECM 型分解；将 `ez-rsa` 改为双端 bit DFS；将 `hard-rsa` 改为 generalized Wiener 与 `phi_6(N)`；将 `SU_RSA` 改为小 `d` 加 `p+q` 高位泄露的二元小根；将 `math_rsa` 改为代数恒等式恢复 `phi`。
- 同步补充 `rsa-specialized-structures-and-oracles.md` 的 CM trace、小私钥/RSA-like 路由和五个 raw 案例索引。
- 本轮不修改 raw 正文，只修正 Crypto 首轮 family 的 raw 案例概括、下一跳和 RSA family 的识别边界。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-crypto-rsa-bucket-retune-20260706-goal.zip`。

## 2026-07-06 — Crypto Hash/Oracle 桶化案例收敛

- 继续检查 `crypto-parameter-triage-family.md` 中 4 条重复 hash/protocol 描述，逐篇读取 raw 后确认其中只有 `SuanHash` 是直接 hash state collision。
- 将 `OhMyCaptcha` 改为 modular CAPTCHA + 表达式 RSA oracle；将 `Flux` 改为有限域二次递推 + SMT 反求自定义 hash key；将 `SuanHash` 改为 sponge-like rate/capacity 差分碰撞；将 `SuanP01y` 改为 `GF(2)[X]/(X^r-1)` 有理重构。
- 同步补充 `hash-protocol-and-oracle-attacks.md` 的表达式 RSA oracle 入口，以及 `oracles-recurrences-captcha-polyglots.md` 的 modular CAPTCHA 和 finite-field recurrence 入口。
- 本轮不修改 raw 正文，只修正 Crypto 首轮 family 的 raw 案例概括、下一跳和相邻 family 的识别边界。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-crypto-hash-oracle-bucket-retune-20260706-goal.zip`。

## 2026-07-06 — Crypto Number/Lattice 桶化案例收敛

- 继续检查 `crypto-parameter-triage-family.md` 中 3 条 number-theory 泛化描述和 3 条 lattice 泛化描述，逐篇读取 raw 后按首轮信号重路由。
- 将 `Classic` 改为 RSA 高位泄露一元 Coppersmith；将 `Decision` 改为 LWE/随机 chunk 判别；将 `yqs` 改为 LWE primal attack 后接无效曲线小子群；将 `f,l and ag++` 改为高次多项式 resultant/快速插值；将 `superguess++` 改为低泄露 HNP lattice sieving；将 `SU_Lattice` 改为高位截断 LFSR 未知模数恢复。
- 同步补充 `lattice-and-lwe.md` 的 truncated recurrence 与低泄露 HNP 路由，以及五个 raw 案例索引。
- 本轮不修改 raw 正文，只修正 Crypto 首轮 family 的 raw 案例概括、下一跳和 lattice family 的识别边界。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-crypto-number-lattice-buckets-retune-20260706-goal.zip`。

## 2026-07-06 — Misc Restricted Shell 桶误归类修正

- 继续检查 `misc-cross-category-triage-family.md` 中 4 条重复“受限 shell/源码后门”描述，逐篇读取 raw 后确认其中 3 条是 C2/PCAP/恶意样本取证，1 条是 SSH 横向与 Git/备份凭据链。
- 将 `ezssh` 改为 Debian OpenSSL 弱 key、Git dangling `.env`、SFTP-only 备份 unit 和内部 API 串联；将两道 Asgard 改为 AES C2/PCAP artifact 取证；将 `vnshell` 改为 WebShell-stage1-stage2-C2-ZipCrypto 多阶段流量分析。
- 同步补充 `malware-c2-session-key-and-protocol-recovery.md` 与 `linux-git-browser-and-container-forensics.md` 的 raw 案例索引，并更新 Misc 路线计数。
- 本轮不修改 raw 正文，只修正 Misc 首轮 family 的 raw 案例概括、下一跳和相邻 family 的识别边界。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-misc-restricted-shell-bucket-reroute-20260706-goal.zip`。

## 2026-07-06 — Misc Python/Agent 桶误归类修正

- 继续检查 `misc-cross-category-triage-family.md` 中 3 条重复 Python/agent/sandbox 描述，逐篇读取 raw 后确认只有传统 CPython audit hook 仍应留在 `pyjails.md`。
- 将 `∀gent` 改为 Node dotted path 原型污染到 compat eval；将 `Shiori` 改为 EXIF 参数驱动的 PNG 像素网格抽样；将 `HuntingAgent` 改为 LLM agent 调度/prompt leak 与 Node VM escape 组合。
- 同步补充 `node-and-prototype.md`、`llm-attacks.md` 和 `image-bitplane-qr-and-jpeg-stego.md` 的 raw 案例索引，并更新 Misc 路线计数。
- 本轮不修改 raw 正文，只修正 Misc 首轮 family 的 raw 案例概括、下一跳和相邻 family 的识别边界。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-misc-python-agent-bucket-reroute-20260706-goal.zip`。

## 2026-07-06 — Misc Matrix/Protocol 桶误归类修正

- 继续检查 `misc-cross-category-triage-family.md` 中 2 条重复矩阵/递推/协议描述，逐篇读取 raw 后确认二者首轮信号不同。
- 将 `Invest on Matrix` 改为 QR 小块重组，不再停留在矩阵/约束求解；将 `ezProtocol` 改为自定义 packet、XOR payload、CRC32 和同连接认证状态复用。
- 同步补充 `image-bitplane-qr-and-jpeg-stego.md` 与 `oracles-recurrences-captcha-polyglots.md` 的 raw 案例索引，并更新 Misc 路线计数。
- 本轮不修改 raw 正文，只修正 Misc 首轮 family 的 raw 案例概括、下一跳和相邻 family 的识别边界。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-misc-matrix-protocol-bucket-reroute-20260706-goal.zip`。

## 2026-07-06 — Misc File Triage 桶误归类修正

- 继续检查 `misc-cross-category-triage-family.md` 中 4 条重复“轻量附件/压缩包/事件材料首检”描述，逐篇读取 raw 后确认它们都不应停留在 file-triage。
- 将 `REDACTED` 改为 PDF 脱敏/字体 CMap/增量保存取证；将 `Vidar Token` 改为链上 ERC-20 异常字段与前端 WASM 解密联动；将 `Artifact Online` 改为符文映射和 cube 状态搜索；将 `ez-iot` 改为 Xtensa ESP 固件逆向与 ESP-NOW 流量重组。
- 同步补充 `pdf-png-gif-and-text-stego.md`、`blockchain-smart-contract-exploitation.md`、`game-state-websocket-and-wasm.md`、`hardware-isa-bootloader-and-kvm.md` 和 `pcap-protocol-credential-recovery-family.md` 的识别边界或 raw 案例索引，并更新 Misc 路线计数。
- 本轮不修改 raw 正文，只修正 Misc 首轮 family 的 raw 案例概括、下一跳和相邻 family 的识别边界。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-misc-file-triage-bucket-reroute-20260706-goal.zip`。

## 2026-07-06 — Misc Encoding 桶误归类修正

- 继续检查 `misc-cross-category-triage-family.md` 中 4 条重复“编码链/助记词/签到/日期文本变换”描述，逐篇读取 raw 后确认只有 `special day` 和 `打好基础` 是纯编码/文本变换。
- 将 `514` 改为 Koishi/Puppeteer `file://` 截图渲染上下文下的 HTML/JS 注入；将 `mymnemonic` 改为图像 bitstream 提取后接 BIP39 `ENT/checksum/seed` 恢复。
- 同步补充 `encodings-qr-and-esolangs.md`、`image-bitplane-qr-and-jpeg-stego.md`、`exotic-secret-sharing-rabin-and-polynomials.md`、`web-first-pass-triage-and-chain-patterns.md` 和 `xss-dom-and-browser-tricks.md` 的 raw 案例索引或分支边界，并更新 Misc 路线计数。
- 本轮不修改 raw 正文，只修正 Misc 首轮 family 的 raw 案例概括、下一跳和相邻 family 的识别边界。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-misc-encoding-bucket-reroute-20260706-goal.zip`。

## 2026-07-06 — Misc Game State 桶误归类修正

- 继续检查 `misc-cross-category-triage-family.md` 中 3 条重复“游戏/交互状态或资源包可控”描述，逐篇读取 raw 后确认 `MirrorBus9` 不应归在游戏状态。
- 将 `master of album` 改为 Socket.IO 限时专辑问答和媒体识别自动化；将 `MirrorBus9` 改为 `F_65521` 仿射协议学习、CHAL/PROVE oracle 和 16-bit checksum 穷举；将 `Minecraft` 改为 Velocity/子服/LuckPerms/PostgreSQL 权限链与 Paper/JDK Log4Shell 组合。
- 同步补充 `game-state-websocket-and-wasm.md`、`oracles-recurrences-captcha-polyglots.md`、`pentest-attack-chains-and-tunneling.md` 和 `known-cves-and-n-day-exploits.md` 的 raw 案例索引或分支边界，并更新 Misc 路线计数。
- 本轮不修改 raw 正文，只修正 Misc 首轮 family 的 raw 案例概括、下一跳和相邻 family 的识别边界。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-misc-game-state-bucket-reroute-20260706-goal.zip`。

## 2026-07-06 — AI/ML 模型案例误归类修正

- 检查 `ml-model-inference-extraction-and-weight-analysis.md` 中 2 条重复“模型输入或特征边界可控”描述，逐篇读取 raw 后确认二者都不属于 adversarial example。
- 新增 `transformer-logit-inversion.md` 承接 `Residue`：已知 GPT-2 权重和目标 logits 时逐 token 枚举并用 KV cache 加速原像恢复。
- 新增 `linear-model-input-lattice-recovery.md` 承接 `babyAI`：从 PyTorch 权重展开无激活网络，构造带小噪声的模线性方程并用 LLL/Babai 恢复输入 flag。
- 更新 AI/ML family 案例表和 `index.md` 的 technique 索引；本轮不修改 raw 正文。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-ai-ml-model-case-reroute-20260706-goal.zip`。

## 2026-07-06 — ctf-ai-ml Skill 直链同步

- 根据 AI/ML 模型案例误归类修正，同步更新 `C:/Users/LMY/.agents/skills/ctf-ai-ml/SKILL.md` 的高频 wiki 入口。
- 将旧的 `model-attacks.md` 直链替换为 `ml-model-inference-extraction-and-weight-analysis.md`，并补入 `linear-model-parameter-recovery.md`、`linear-model-input-lattice-recovery.md` 和 `transformer-logit-inversion.md`。
- 增加线性网络展开为模线性方程时的 pivot 提示：先判断模型语义，若核心只剩格构造再转 `ctf-crypto`。
- 本轮不修改 raw 正文。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-ctf-ai-ml-skill-link-sync-20260706-goal.zip`。

## 2026-07-06 — Tooling 页边界文本收敛

- 检查全部 `*-tooling.md` 后确认旧的机械化“复用重点”占位句已基本清除，但多页仍保留“知识页覆盖状态/后续补强方向”式维护清单，和 tooling 页的工具入口职责不够贴合。
- 将 AI/ML、Crypto、Forensics、Malware、Misc、OSINT、Pentest、Pwn、Reverse、Web 工具页顶部统一收敛为“工具选择边界”，并按方向写明入口选择、不应进入该工具链的情况和补工具经验的具体触发条件。
- 补查不符合 `*-tooling.md` 命名模式的 3 个 Reverse 专项工具页：`disassemblers-debuggers-and-basic-tools.md`、`frida-angr-lldb-and-x64dbg.md`、`qiling-triton-pin-and-ldpreload.md` 已有明确工具路由和失败转向，本轮不改正文。
- 顺手把 AI/ML 模型 family 中“最小样本集”改成“少量代表性查询样本”，避免和旧 tooling 模板措辞混淆。
- 保留各 tooling 页原有工具路径、调用方式和失败转向；本轮不新增工具、不修改 raw 正文，也不改变 `index.md` 的页面入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-tooling-boundary-rewrite-20260706-goal.zip`。
- 补查前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-reverse-tooling-audit-and-ai-family-phrase-20260706-goal.zip`。

## 2026-07-06 — Technique 边界保留审计

- 继续按健康检查顺序抽查 technique 页，重点检查 Reverse/Malware、AI/ML、Web/browser 和网络协议类页面是否应合并、拆分、重命名或删除。
- 保留 `embedded-python-pyd-custom-aes.md` 与 `compare-breakpoint-plaintext-recovery.md` 的拆分：前者核心是嵌入式 Python/PYD loader 与 AES-family 变体复原，后者核心是最终比较点动态读取内置明文。
- 保留 `powershell-staged-payload-and-clipboard-phishing.md`、`vmp-client-server-smc-rc4-recovery.md`、`windows-kernel-ioctl-hidden-feedback-maze.md`：三者分别以脚本 stage 静态还原、受保护 client/server 协议动态恢复、Windows driver IOCTL 隐藏反馈探图为首轮信号，工具链和失败 pivot 不同。
- 保留 `linear-model-parameter-recovery.md`、`linear-model-input-lattice-recovery.md`、`transformer-logit-inversion.md` 的拆分：参数恢复、输入向量格恢复和 logits 原像搜索的最小证据与验证方式不同。
- 保留 `bgp-rpki-route-hijack.md` 和 `csp-xsleak-and-browser-exfiltration.md` 作为独立 technique：前者是 BGP/RPKI 路由控制与 origin 校验边界，后者是浏览器侧 oracle 外带。
- 本轮 technique 边界审计未发现需要修改正文的合并/删除项；仅记录保留取舍，不修改 raw 正文，也不改变 `index.md`。
- 审计前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-technique-boundary-audit-log-20260706-goal.zip`。

## 2026-07-06 — 高密度 Family 页统计口径修正

- 抽查 `web-first-pass-triage-and-chain-patterns.md`、`reverse-first-pass-workflow-and-debugging.md` 和 `misc-cross-category-triage-family.md`，确认三者仍有首轮路由、失败 pivot 和 case routing 价值，暂不拆分或删除。
- 校验三页内部 active 链接均存在；`misc-cross-category-triage-family.md` 的 58 篇 raw 案例声明与案例表链接数一致。
- 发现 Misc 路线表原“WP 数”列统计的是“raw 到下一跳”的多标签触达关系，列值相加会超过唯一 raw 篇数；将列名改为“案例触达”并补充说明，避免误读为唯一 WP 篇数。
- 本轮不修改 raw 正文，也不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-misc-route-count-label-20260706-goal.zip`。

## 2026-07-06 — 外设 Capture 与物理信号 Family 边界修正

- 抽查短 family 页时发现 `peripheral-capture.md` 与 `keyboard-mouse-audio-and-physical-puzzles.md` 都覆盖键盘、鼠标、LED、音频/视频等信号，存在未来误归类风险。
- 保留二者拆分：`peripheral-capture.md` 负责 USB/Bluetooth/URB/HID report 解析和事件重组；`keyboard-mouse-audio-and-physical-puzzles.md` 负责解析后的动作轨迹、频谱、视频和物理时序语义还原。
- 分别补充边界说明和下一跳转向，不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-peripheral-physical-boundary-20260706-goal.zip`。

## 2026-07-06 — 文件首检 Family 边界修正

- 抽查 `file-signatures-and-flag-artifact-hunting.md` 与 `file-triage-archives-and-one-liners.md`，确认二者都应保留为 family，但边界需要更清楚。
- 明确 `file-signatures-and-flag-artifact-hunting.md` 负责 Forensics 文件内部 artifact、metadata、尾随数据、隐藏对象和 magic 修复。
- 明确 `file-triage-archives-and-one-liners.md` 负责 Misc 轻量附件、签到、一行式命令、hash/cipher 识别和 API 枚举的首轮分流；一旦进入文件内部 artifact 或复杂归档恢复，转到对应 Forensics family。
- 补充两页互链和路由表下一跳；本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-file-triage-boundary-20260706-goal.zip`。

## 2026-07-06 — Forensics 存储恢复 Family 边界修正

- 抽查 `disk-memory-vm-and-container-forensics.md`、`filesystems-memory-dumps-and-raid.md`、`filesystem-archive-recovery-and-repair.md`，确认三者都仍有独立 family 价值。
- 明确三层分工：`disk-memory-vm-and-container-forensics.md` 是载体入口和流程选择；`filesystems-memory-dumps-and-raid.md` 是分区、RAID、VMDK sparse、minidump/core dump 和内存 key carving 等底层恢复；`filesystem-archive-recovery-and-repair.md` 是文件/归档层删除恢复、损坏压缩包、ZipCrypto 和结构化文件恢复。
- 补充交叉转向和边界说明，避免把可挂载文件系统内的归档修复误归到底层恢复，也避免在载体未确定时直接进入归档恢复。
- 本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-forensics-storage-family-boundary-20260706-goal.zip`。

## 2026-07-06 — Crypto/Malware/Pwn Family 保留审计

- 抽查 Crypto 轻量/流密码/oracle/分组模式聚类，确认 `classical-xor-and-substitution-ciphers.md`、`rc4-lfsr-and-keystream-reuse.md`、`hash-protocol-and-oracle-attacks.md`、`block-mode-misuse-family.md` 的分工清楚：分别处理古典与简单 XOR、可建模 keystream 状态、hash/protocol oracle、分组模式和 MAC 边界。
- 抽查 Malware 聚类，确认 `scripts-and-obfuscation.md` 是载荷链分阶段还原 family，`pe-and-dotnet.md` 是样本本体分析 family，`malware-c2-session-key-and-protocol-recovery.md` 是 C2 会话密钥与协议恢复 technique，暂不合并。
- 抽查 Pwn ROP/runtime 聚类，确认 `ret2csu-dynelf-and-shellcode.md`、`stack-pivots-srop-and-seccomp-rop.md`、`seccomp-ret2dlresolve-and-runtime-primitives.md`、`runtime-protection-and-tls-exploits.md` 的首轮问题不同，暂不合并或拆分。
- 本轮这些页面不改正文，仅记录保留判断；不修改 raw 正文，不调整 index 入口。
- 审计前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-crypto-malware-pwn-boundary-audit-log-20260706-goal.zip`。

## 2026-07-06 — Web Auth 与 OAuth Family 边界修正

- 抽查 `auth-bypass-cookies-and-hidden-routes.md`、`auth-jwt.md`、`auth-edge-cases-and-protocol-bypasses.md`、`oauth-saml-cors-and-cicd.md`，确认认证聚类整体应保留拆分。
- 明确 `auth-bypass-cookies-and-hidden-routes.md` 是应用本地授权入口；若信任边界落在 OAuth/OIDC/SAML、CORS、Git/CI runner、外部 IdP 或平台 token，应转 `oauth-saml-cors-and-cicd.md`。
- 在 `oauth-saml-cors-and-cicd.md` 中补充跨系统信任边界说明，并把原先纯文字的 `state` 缺失、登录页投毒下一跳改为可点击页面。
- 本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-web-auth-oauth-boundary-20260706-goal.zip`。

## 2026-07-06 — Web 服务端解释链边界修正

- 抽查 `php-lfi-ssti-ssrf-and-type-juggling.md`、`path-traversal-ssrf-upload-and-rsc.md`、`ruby-php-upload-and-ssti-rce.md`、`sqli-upload-deser-and-command-rce.md`，确认它们分别承担解释层差异、资源定位/上传/内部服务链、执行面确认、入口到 RCE 链路升级的 family 角色。
- 补充 `php-lfi-ssti-ssrf-and-type-juggling.md` 与 `ruby-php-upload-and-ssti-rce.md` 的互相转向边界：前者处理 LFI/SSRF/XXE/弱比较/格式化器差异是否成立，后者处理已进入 eval、模板求值、命令拼接、WebShell 或可执行上传后的 RCE 落地。
- 本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-web-server-execution-boundary-20260706-goal.zip`。

## 2026-07-06 — Web Parser/Node/Workflow 链路入链修正

- 抽查 `parser-wrapper-and-legacy-ssrf-tricks.md`、`polyglot-url-tricks-and-ssrf-leaks.md`、`path-confusion-to-signed-internal-request-chain.md`、`node-and-prototype.md`、`workflow-runner-internal-api-chain.md`、`protocol-relay-and-internal-service-injection.md` 和 `artifact-trust-ssrf-to-node-require-rce.md`。
- 确认 parser/polyglot/node 继续作为 family，path-confusion/protocol-relay/artifact/workflow 继续作为 technique；它们的首轮证据、工具链和失败 pivot 不同，暂不合并或删除。
- 补充 parser family 到 path-confusion technique 的下一跳，补充 Node family 到 workflow runner technique 的下一跳，并补齐 path-confusion/workflow 的相邻技巧入链。
- 本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-web-parser-node-workflow-boundary-20260706-151348.zip`。

## 2026-07-06 — AI/ML 与 Crypto 低入链页面支撑复核

- 复核 AI/ML 低入链 technique：`linear-model-parameter-recovery.md`、`linear-model-input-lattice-recovery.md` 和 `transformer-logit-inversion.md` 分别由 SU 参数恢复、`SU_babyAI` 和 Residue logits 反演 raw 支撑，首轮证据与验证方式不同，暂不合并。
- 保留 `adversarial-ml.md` 为 family：其 raw 覆盖 FGSM/PGD/C&W、patch、poisoning 和 backdoor，多路线分流价值高于压成单一 technique；后续有更多 WP 时再考虑拆具体 technique。
- 复核 Crypto 低入链页面时发现 `crypto-parameter-triage-family.md` 已把 `D3CTF2022-d3share-wp` 指向 `exotic-secret-sharing-rabin-and-polynomials.md`，但 exotic 页自身缺少该 raw 的案例入口。
- 补充 `exotic-secret-sharing-rabin-and-polynomials.md` 的扰动多项式 key sharing 路由、`D3CTF2022-d3share-wp` 案例、`VNCTF2026-mymnemonic-wp` 显式 raw 引用，以及到 `lattice-and-lwe.md` 的相邻技巧。
- 本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-exotic-crypto-family-case-link-20260706-152118.zip`。

## 2026-07-06 — Blockchain Exploitation 与交易取证边界修正

- 抽查 `blockchain-smart-contract-exploitation.md` 与 `blockchain-and-transaction-forensics.md`，确认二者不应合并：前者负责合约状态机、账户绑定、capability/resource 和链上调用约束，后者负责 TXID/address/UTXO/金额流转取证。
- 补齐 `blockchain-smart-contract-exploitation.md` 的显式 raw 来源清单，使案例表中的 Solana、Anchor、Sui、ERC-20/WASM、EVM 伪随机和 settle 重放案例都能从“原始资料”定位回 raw。
- 在交易取证 technique 中补充转向：一旦目标变成修改合约状态、mint/claim/settle 或利用 proxy/PDA/Move resource，应转 blockchain exploitation family。
- 本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-blockchain-family-forensics-boundary-20260706-152318.zip`。

## 2026-07-06 — Windows Driver 与平台 Family 案例归位

- 抽查 `windows-kernel-ioctl-hidden-feedback-maze.md`、`mobile-firmware-kernel-and-game-re.md` 和相邻 loader/signal family 后，确认 `windows-kernel-ioctl-hidden-feedback-maze.md` 应保持为具体 technique，不承担所有 Windows driver / WoW64 案例入口。
- 将 `HGAME2026-noncesense-wp`、`D3CTF2025-d3kernel-wp` 和 `D3CTF2022-d3w0w-wp` 的案例索引从具体 technique 移到 `mobile-firmware-kernel-and-game-re.md`，分别归为 client/driver IOCTL 协议、R3/R0 反调试 + DeviceIoControl VM、WoW64 位宽切换隐藏逻辑。
- 在 `mobile-firmware-kernel-and-game-re.md` 增加 WoW64 / Heaven's Gate / 模式切换路由，并补充到 `signal-trace-and-packed-anti-analysis.md` 的相邻入口。
- 本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-windows-driver-case-rehome-20260706-152559.zip`。

## 2026-07-06 — AI/ML 模型 Family 原始资料清单补齐

- 只读扫描发现 `ml-model-inference-extraction-and-weight-analysis.md` 的案例表已引用 6 篇 AI/ML raw WP，但末尾 `原始资料` 清单只列基础 `model-attacks.md`。
- 补齐该 family 的 frontmatter `raw:` 和 `原始资料`，纳入 Residue、三篇 SU 线性参数恢复、`SU_babyAI` 和 `SU_easyLLM`；具体 technique 归属不变。
- 本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-ai-ml-family-raw-list-complete-20260706-152754.zip`。

## 2026-07-06 — Hardware/ISA Family 原始资料清单补齐

- 复核 `hardware-isa-bootloader-and-kvm.md` 的案例表，确认 `ACTF2026-acpu-wp`、`ACTF2026-amcu-wp`、`LilacCTF2026-justrom-wp` 和 `VNCTF2026-ez-iot-wp` 都直接支撑 hardware/ISA/firmware family。
- 补齐该页 frontmatter `raw:` 和 `原始资料`，使 CPU 瞬态执行、MCU 固件外设、ROM/MMIO 和 ESP-NOW 固件流量案例都能从页面末尾定位回 raw。
- 本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-hardware-isa-raw-list-complete-20260706-152928.zip`。

## 2026-07-06 — OSINT Account Family 原始资料清单补齐

- 复核 `osint-account-public-media-correlation.md` 的案例表，确认 `SU_CyberTrackWP`、`RCTF2025-speak-softly-love-wp`、`RCTF2025-wanna-feel-love-wp` 和 `SU_SigninWP` 都支撑公开账号、媒体、archive、团队页和身份链 pivot。
- 补齐该页 frontmatter `raw:` 和 `原始资料`，避免案例表引用的 raw 只存在正文中而末尾来源清单不可追踪。
- 本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-osint-account-raw-list-complete-20260706-153057.zip`。

## 2026-07-06 — Encodings Family 原始资料清单补齐

- 复核 `encodings-qr-and-esolangs.md` 的案例表，确认 `ACTF2026-special-day-wp` 和 `HGAME2026-打好基础-wp` 是纯编码/基础变换案例。
- `SU_Artifact_OnlineWP` 只支撑符文到明文替换映射这一段；后续 cube 状态搜索仍由 `game-state-websocket-and-wasm.md` 承接。
- 补齐该页 frontmatter `raw:` 和 `原始资料`，不改变页面分类和下一跳。
- 本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-encodings-raw-list-complete-20260706-153229.zip`。

## 2026-07-06 — Game State Family 原始资料清单补齐

- 复核 `game-state-websocket-and-wasm.md` 的案例表，确认 `SU_Artifact_OnlineWP`、`ACTF2026-master-of-album-wp` 和 `VNCTF2026-minecraft-wp` 都直接支撑游戏状态、WebSocket/Socket.IO 或游戏服务边界分流。
- `SU_Artifact_OnlineWP` 在 encodings family 中只承担符文映射案例，在本页承担 cube 状态搜索和服务端执行点案例；同一 raw 因支撑两个不同模式而双向引用。
- `ACTF2026-master-of-album-wp` 支撑 Socket.IO 自动答题与弱 session 绑定，`VNCTF2026-minecraft-wp` 支撑 Minecraft 多服/插件/权限链入口，后续 CVE 利用仍转 Pentest/CVE 页面。
- 补齐该页 frontmatter `raw:` 和 `原始资料`，不改变页面分类和下一跳。
- 本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-game-state-raw-list-complete-20260706-153420.zip`。

## 2026-07-06 — Race Condition Technique 原始资料清单补齐

- 复核 `race-condition-and-concurrency-exploits.md` 的案例表，确认 `HGAME2026-producer-and-comsumer-wp`、`NCTF2026-quantum-vault-wp` 和 `RCTF2025-ultimatefreeloader-wp` 都直接支撑并发窗口、TOCTOU 或锁粒度冲突。
- 保留该页为跨 Pwn/Web/Pentest/Misc 的 technique：这些案例共享“检查/使用分离或两个业务流未互斥”的最小证据，差异体现在后续落地点而不是首轮判断。
- 补齐该页 frontmatter `raw:` 和 `原始资料`，不改变关联页面和 index 入口。
- 本轮不修改 raw 正文。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-race-condition-raw-list-complete-20260706-153620.zip`。

## 2026-07-06 — Forensics Cross-Domain 来源与下一跳修正

- 复核 `cross-domain-forensics-technique-map.md` 的 WP 案例表，发现 `SU_LightNovelWP` 被误概括成文档/媒体隐写；raw 实际是 AD/Kerberos/NTLM、MSRPC 与 TSCH 远程任务调度流量取证。
- 修正 `SU_LightNovelWP` 在 cross-domain family 的复用信号和下一跳，并在 `pcap-protocol-credential-recovery-family.md` 增加 Kerberos/NTLM + MSRPC/TSCH 路由和案例入口。
- `SU_forensicsWP` 直接支撑 Windows artifact 综合分流，补入 `windows-registry-logs-and-credentials.md` 的案例索引和来源清单。
- 补齐三页 frontmatter `raw:` 和 `原始资料`；本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-forensics-cross-domain-source-routing-20260706-153900.zip`。

## 2026-07-06 — Geolocation Family 案例边界修正

- 复核 `geolocation-and-media.md` 的案例表，确认 `LilacCTF2026-sky-is-ours-wp` 是航班/地点/日期交叉定位案例，直接支撑本页。
- `SU_CyberTrackWP` 的核心是公开账号、Gravatar、NameMC、社交平台和 Discord 身份链闭合，已由 `osint-account-public-media-correlation.md` 承接；从 geolocation family 案例表移除，避免把账号 OSINT 误挂为地理定位。
- 补齐 `LilacCTF2026-sky-is-ours-wp` 的 frontmatter `raw:` 和 `原始资料`；本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-geolocation-case-boundary-20260706-154130.zip`。

## 2026-07-06 — LLM Attacks Family 分支与来源补齐

- 复核 `llm-attacks.md` 的案例表，确认 `RCTF2025-the-alchemists-cage-wp` 是隐藏系统提示/secret 泄露案例，`SU_easyLLMWP` 是 LLM 低温输出被当作 AES key 材料后的采样碰撞案例。
- 在 LLM 路由表中补充“LLM 输出作为 key/seed/password”分支，避免把 `SU_easyLLMWP` 简化成普通 prompt injection；并增加到 `block-mode-misuse-family.md` 的关联入口。
- 补齐该页 frontmatter `raw:` 和 `原始资料`；本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-llm-attacks-raw-branch-complete-20260706-154310.zip`。

## 2026-07-06 — 小页面 Raw 来源闭合补齐

- 复核剩余单项漏链页面后，确认 `SU_chaosWP` 支撑 ZipCrypto/AVIF 已知明文恢复，补入 `filesystem-archive-recovery-and-repair.md`。
- `SU_MirrorBus9WP` 支撑有限域仿射状态机、固定 challenge 和 16-bit oracle 爆破，补入 `oracles-recurrences-captcha-polyglots.md`。
- `VNCTF2026-ez-iot-wp` 同时支撑硬件固件逆向和 ESP-NOW 流量重组，补入 `pcap-protocol-credential-recovery-family.md` 来源清单；`HGAME2026-redacted-wp` 补入 PDF redaction family。
- `VNCTF2026-minecraft-wp` 支撑 Pentest attack graph 中的游戏代理、子服、共享权限库和 Log4Shell 链，补入 `pentest-attack-chains-and-tunneling.md` 并增加到 game-state family 的相邻入口。
- `RCTF2025-514-wp` 支撑 Puppeteer `file://` 渲染器和 bot 截图回显，补入 `xss-dom-and-browser-tricks.md`。
- 本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-small-raw-source-list-complete-20260706-154610.zip`。

## 2026-07-06 — Crypto ZKP Family 入链修正

- 低入链 family 扫描发现 `zkp-secret-sharing-and-proof-systems.md` 只有 blockchain exploitation 入链，但它实际是 Crypto 首轮参数分流中的证明系统、约束电路、commitment、share 和 pairing oracle 下一跳。
- 保留该页为 family，不改成 technique：ZKP、garbled circuit、Shamir、KZG 和阈值签名的第一步判断不同，但共同点都是 verifier/share/proof 关系缺约束。
- 在 `crypto-parameter-triage-family.md` 的参数分流表和关联技巧中补入 ZKP/proof-system 入口；本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-crypto-zkp-family-inlink-20260706-154920.zip`。

## 2026-07-06 — 低入链、Tooling 与多页引用复核结论

- 复扫低入链 technique 后，仅剩 `linear-model-input-lattice-recovery.md`、`transformer-logit-inversion.md` 和 `lorenz-and-book-cipher-attacks.md` 三个候选；前两者已在 AI/ML 复核中确认分别由输入格恢复与 logits 反演 raw 支撑，后者由 classical crypto family 入链且解法骨架涉及 Lorenz delta、Baudot shift 和 book cipher distance，暂不合并。
- 复扫低入链 family 后无剩余候选；`zkp-secret-sharing-and-proof-systems.md` 已通过 Crypto triage 入链修复。
- 抽查 13 个 tooling 页，确认均有工具选择边界、环境/路径/调用层和失败转向；严格占位文本扫描结果为 0，暂不再改 tooling 正文。
- 复核多页引用最高的 `WMCTF2025-videoplayer-wp` 与 `WMCTF2025-want2become-magicalgirl-wp`：二者都是多阶段 reverse raw，分别支撑 VMP/反调试、dump、运行时 patch、媒体导出，以及 Android/Flutter 平台、反 hook、trace 和自解密算法恢复；保留多页引用，不做删链。
- 本轮最终结构校验：active Markdown 断链 0，wiki 类型错误 0，index 漏页 0，backup 非 zip 0，wiki 非 Markdown 条目 0，小页面 raw 来源清单漏项 0，占位机械文本 0。
- 本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-health-audit-decision-log-20260706-155240.zip`。

## 2026-07-06 — Raw 引用覆盖统计

- 对 `raw/<direction>/*.md` 与 `wiki/*.md` 的 raw 链接做覆盖统计，所有 raw Markdown 均至少被一个 wiki 页面引用：ai-ml 9/9、crypto 84/84、forensics 19/19、malware 3/3、misc 71/71、osint 5/5、pentest 2/2、pwn 92/92、reverse 117/117、web 102/102。
- 该统计只证明“有入口”，不等同于每个概括都已逐字复核；语义准确性仍以逐页 raw 抽查和 family/technique 边界检查为准。
- 本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-raw-coverage-health-log-20260706-155520.zip`。

## 2026-07-06 — 元数据与 Family 边界继续复核

- 扫描发现 `kaslr-kpti-smep-and-kernel-debugging.md`、`known-cves-and-n-day-exploits.md`、`path-traversal-ssrf-upload-and-rsc.md` 和 `packers-deobfuscation-and-debug-automation.md` 的正文“原始资料”已列出案例 raw，但 frontmatter `raw:` 未同步；逐页抽查后确认这些案例仍支撑当前页面，补齐元数据。
- `WMCTF2025-wm-easyker-wp`、`WMCTF2025-wm-easynetlink-wp` 和 `WMCTF2025-wmkpf-wp` 都支撑 kernel KASLR/KPTI/SMEP/SMAP 保护绕过、调试和最终落点选择。
- `WMCTF2025-shopping-company-phishing-email-wp` 支撑 Open WebUI N-day 和 AI 工具链 CVE/N-day 串联，`VNCTF2026-minecraft-wp` 支撑 Paper/OpenJDK 版本边界下的 Log4Shell 链。
- `WMCTF2025-pdf2text-wp` 与 `WMCTF2025-rustdesk-change-client-backend-wp` 都支撑“可控资源定位交给后端解释”的路径穿越、解析器加载或内部服务注入路线；`WMCTF2025-videoplayer-wp` 支撑 VMP/OEP dump 后仍需业务逻辑 trace 的 packer/deobfuscation 分流。
- 为 `ecc-dlp-and-signature-attacks.md`、`mt-lcg-and-seed-recovery.md` 和 `prng-z3-lcg-and-timing-attacks.md` 补充合并/拆分结论，明确它们保留为 family 的理由和相邻页边界。
- 本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-metadata-and-family-boundary-audit-20260706-continued.zip`。

## 2026-07-06 — Family Pivot 与 Raw 图片链路复核

- 复扫 family 页出链与分流文本，`family_pivot_candidates=0`；当前 family 页均有足够相邻页面入口和路由/分流语义，没有发现只剩目录作用的 family。
- 对 raw Markdown 的本地链接做只读检查，唯一命中是 `VNCTF2026-markdown2world-wp.md` fenced code block 中的示例 payload `![a](<目标 .so 或本地资源路径>)`，不是实际附件断链。
- 对 raw 图片链接和 WP 图片目录命名做规范化扫描，排除 `./目录名/...` 正常相对路径和 fenced code 示例后，`raw_image_link_issues_normalized=0`。
- 本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-family-raw-image-validation-log-20260706-continued.zip`。

## 2026-07-06 — Web First-Pass 案例下一跳复核

- 抽查 `web-first-pass-triage-and-chain-patterns.md` 中 `WMCTF2025-pdf2text-wp`、`HGAME2026-my-monitor-wp`、`D3CTF2019-ezupload-wp` 和 `D3CTF2025-d3model-wp` 的 raw 证据，修正下一跳归属。
- `pdf2text` 同时支撑路径穿越、Python pickle 反序列化和解析器二段加载；首轮下一跳补入 `path-traversal-ssrf-upload-and-rsc.md`，保持反序列化与 parser-wrapper 入口。
- `my-monitor` 的关键不是认证协议绕过，而是 `sync.Pool` 对象复用/错误路径残留字段污染，后续才进入 `bash -c` 命令解释；下一跳改为并发/对象复用与命令注入。
- `ezupload` 兼具 PHP unserialize 文件写与 glob 路径定位/.htaccess 上传解析链，补入反序列化 family；`d3model` 明确基于 CVE-2025-1550，补入 CVE/N-day family，并在反序列化 family 保留模型格式加载案例。
- 本轮同步补齐目标页的案例表和 frontmatter `raw:`，不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-web-first-pass-case-routing-audit-20260706-continued.zip`、`D:/文档/markdown文件/ctf-wiki/backups/pre-web-first-pass-target-case-routing-20260706-continued.zip`。

## 2026-07-06 — Reverse eBPF 与内核侧路由修正

- 抽查 `reverse-first-pass-workflow-and-debugging.md` 中 eBPF/内核侧案例，发现 `Bugku-Berkeley-wp` 只跳到 tooling 页，弱化了“真实校验在内核侧 instrumentation”的 technique/family 归属。
- 将 `Bugku-Berkeley-wp` 下一跳改为 `loader-vm-image-and-kernel-patterns.md` 与 `mobile-firmware-kernel-and-game-re.md`；`ACTF2026-abyssgate-wp` 也补入平台/内核边界下一跳，避免只停留在 loader 视角。
- 在 `loader-vm-image-and-kernel-patterns.md` 增加 eBPF/loader 案例表，补入 `ACTF2026-abyssgate-wp` 和 `Bugku-Berkeley-wp`；在 `mobile-firmware-kernel-and-game-re.md` 增加对应平台边界案例。
- 本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-reverse-ebpf-kernel-routing-20260706-continued.zip`。

## 2026-07-06 — Reverse 基础 Tooling-only 案例复核

- 扫描 `reverse-first-pass-workflow-and-debugging.md` 中完全指向 tooling 页的案例，仅剩 `Bugku-week1_re1-wp`、`Bugku-week2_re1-wp` 和 `RCTF2025-chaos-wp`。
- `Bugku-week1_re1-wp` 是明文字符串搜索，`RCTF2025-chaos-wp` 是直接运行输出；二者保留在 `disassemblers-debuggers-and-basic-tools.md`，不强行提升为 technique。
- `Bugku-week2_re1-wp` 的可复用价值不是工具本身，而是按真实倒序循环模拟静态数组的原位变换；补入 `self-decrypting-strings-and-lattice-patterns.md` 并保留基础工具入口。
- 本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-reverse-basic-tooling-case-routing-20260706-continued.zip`。

## 2026-07-06 — Pwn 跨平台与基础溢出承接页补强

- 复核 `pwn-first-pass-red-flags-and-protections.md` 的跨方向下一跳，确认 `hardware-isa-bootloader-and-kvm.md` 已承接 TrustZone、RISC-V CPU 与 MCU 案例。
- `windows-arm-and-cross-platform-exploits.md` 原先只有规则无 WP 案例，补入 `ACTF2026-agpu-wp`、`LilacCTF2026-gate-way-wp` 和 `D3CTF2023-d3op-wp`，覆盖 ARM64 kernel/GPU、Hexagon ROP 与 OpenWrt AArch64 ROP。
- `D3CTF2023-d3op-wp` 的第一漏洞形态仍是协议入口触发的栈溢出，Pwn 首轮下一跳补入 `overflow-basics.md`，并在 overflow family 增加案例入口。
- 本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-pwn-cross-platform-routing-20260706-continued.zip`。

## 2026-07-06 — Crypto Fibonacci RSA 双重归属修正

- 复核 `crypto-parameter-triage-family.md` 中 `Bugku-Fibonacci-RSA-wp`，确认它不是纯数论题：Pisano period 用于恢复 `S`，但 `p = next_prime(S**16)` 直接决定 RSA 分解。
- 将该案例下一跳从单独 `number-theory-and-algebra-attacks.md` 扩展为数论 + `rsa-specialized-structures-and-oracles.md`，并把 RSA 统计从 14 调整为 15。
- 在 RSA specialized 页补充“递推序列/周期和生成 prime”的分支与案例；在 number-theory 页补充 Pisano period/周期求和分支与同一 raw 的数论侧案例。
- 本轮不修改 raw 正文，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-crypto-fibonacci-rsa-routing-20260706-continued.zip`。

## 2026-07-06 — LLM 输出派生 AES Key 路由修正

- 抽查新增 AI/ML technique 时复核 `SU_easyLLMWP` raw，确认该题核心不是普通 prompt injection，而是 `key = SHA256(LLM_output)[:16]`、低温 LLM 输出空间集中和 AES-CBC 密文碰撞验证。
- 修正 `ml-model-inference-extraction-and-weight-analysis.md` 与 `llm-attacks.md` 的案例描述，把下一跳明确到 LLM 输出 key/seed/password 分支，并增加到 `block-mode-misuse-family.md` 的 key derivation pivot。
- `block-mode-misuse-family.md` 增加 LLM/password generator 派生 key 路线、关联入口和 `SU_easyLLMWP` 来源；`index.md` 同步补充 LLM 输出派生 key/seed/password 查询提示。
- 同步 `ctf-ai-ml/SKILL.md` 和 `ctf-crypto/SKILL.md`：首轮判断与高频页说明补入 LLM 输出派生 key/seed/password，避免以后仍按普通 prompt injection 或纯 AES mode 处理。
- 本轮不修改 raw 正文，不新增 wiki 页面。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-llm-output-key-index-routing-20260706-194804.zip`、`D:/文档/markdown文件/ctf-wiki/backups/pre-llm-output-key-skill-sync-20260706-195021.zip`。

## 2026-07-06 — Skill 高频 Wiki 直链漂移修正

- 扫描 `ctf-*` skills 的高频 wiki 绝对路径，发现 `ctf-malware` 仍指向已不存在的 `wiki/c2-and-protocols.md`，`ctf-osint` 仍指向已不存在的 `wiki/social-media.md`。
- 将 Malware C2 高频入口改为 `malware-c2-session-key-and-protocol-recovery.md`，将 OSINT 账号社交入口改为 `osint-account-public-media-correlation.md`；两者均是现有 index 中的 active 页面。
- 本轮不修改 raw 正文，不新增 wiki 页面，只同步 skill 直链。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-skill-wiki-link-drift-sync-20260706-195228.zip`。

## 2026-07-06 — Active Tree 空本地目录清理

- 结构校验发现仓库根目录下存在空的 `.agents/` 与 `.codex/` 本地目录；二者不属于 active tree 的 `raw/`、`wiki/`、`backups/` 或根文档。
- 删除前确认两个目录都在当前工作区内且为空，随后移除，避免 active tree 严格扫描继续误报。
- 本轮不修改 raw 正文，不新增 wiki 页面。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-empty-local-dotdir-cleanup-20260706-195533.zip`。

## 2026-07-06 — Web Family 首轮证据段补强

- 扫描 family 页核心结构时发现 `auth-bypass-cookies-and-hidden-routes.md`、`path-traversal-ssrf-upload-and-rsc.md` 和 `xss-dom-and-browser-tricks.md` 仍偏旧式“变体路由”写法，缺少独立的识别信号或最小证据段。
- 为认证绕过页补充身份字段、代理/应用授权差异、cookie/session/token 和最小请求差异证据；为路径/SSRF/上传页补充可控资源定位进入后端解释层的识别信号。
- 为 XSS/DOM/browser 页补充 payload 落点、bot 触发条件、外带 oracle 和截图/file renderer 最小证据，便于首轮快速判断是浏览器侧问题还是服务端解释链。
- 本轮不修改 raw 正文，不新增 wiki 页面，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-web-family-evidence-sections-20260706-200239.zip`。

## 2026-07-06 — Reverse 平台 Family 首轮证据段补强

- 继续处理旧式 family 页核心结构，补强 `android-games-hardware-and-runtime-platforms.md`、`loader-vm-image-and-kernel-patterns.md` 和 `mobile-firmware-kernel-and-game-re.md` 的独立识别信号与最小证据段。
- Android/游戏平台页明确 UI/资源/native/engine plugin 数据流证据；loader/VM/image/kernel 页明确真实逻辑出现时机、载荷映射和 VM/image 可复算证据。
- Mobile/firmware/kernel 页补充 Mach-O、固件、驱动、eBPF、CAN/UDS 等底层环境的入口、架构、设备接口和最小可重放样本要求。
- 本轮不修改 raw 正文，不新增 wiki 页面，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-reverse-platform-family-evidence-sections-20260706-200410.zip`。

## 2026-07-06 — Crypto Family 首轮证据段补强

- 继续收敛 family 核心结构，为 `classical-xor-and-substitution-ciphers.md` 补充轻量古典/XOR/视觉编码的最小证据要求。
- 为 `hash-protocol-and-oracle-attacks.md` 补充 hash/MAC/protocol oracle 的识别信号和可重放响应差异证据，强调 oracle 层级和校验顺序。
- 为 `rsa-attacks.md` 与 `rsa-specialized-structures-and-oracles.md` 补充基础 RSA 快速排查、长尾结构建模、Coppersmith/oracle/fault/signature 实现 bug 的首轮证据边界。
- 本轮不修改 raw 正文，不新增 wiki 页面，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-crypto-family-evidence-sections-20260706-200535.zip`。

## 2026-07-06 — Web 链路 Family 首轮证据段补强

- 继续收敛旧式 Web family 页结构，为 `php-lfi-ssti-ssrf-and-type-juggling.md` 补充服务端解释层差异识别信号。
- 为 `ruby-php-upload-and-ssti-rce.md` 补充执行面识别信号、最小可验证副作用和常见误判，区分文件落地、模板报错与真正 RCE。
- 为 `sqli-upload-deser-and-command-rce.md` 补充已有 primitive 到执行链升级的识别信号与跨服务证据链要求。
- 为 `web-first-pass-triage-and-chain-patterns.md` 补充最小请求差异、输入解释层、可观测反馈和常见误判，使首轮入口不只是一张路由表。
- 本轮不修改 raw 正文，不新增 wiki 页面，不调整 index 入口。
- 校验修正：补齐 `php-lfi-ssti-ssrf-and-type-juggling.md` 的最小证据段，以及 `web-first-pass-triage-and-chain-patterns.md` 的识别信号段。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-web-chain-family-evidence-sections-20260706-200711.zip`、`D:/文档/markdown文件/ctf-wiki/backups/pre-web-family-evidence-fix-20260706-200946.zip`。

## 2026-07-06 — Reverse 语言与低频载体 Family 证据段补强

- 继续收敛 reverse 旧式 family 页结构，为 `reverse-first-pass-workflow-and-debugging.md` 补充首轮识别信号、最小证据和常见误判，明确何时转语言运行时、VM、平台、crypto 或 misc。
- 为 `go-rust-jvm-and-cpp-reversing.md` 补充语言 runtime、对象布局、动态加载和 native extension 的证据门槛，避免只按文件名或反编译结果判断。
- 为 `python-bytecode-esolangs-and-uefi.md` 补充字节码版本、opcode 表、解释器语义、legacy 格式和最小正向验证样本要求。
- 为 `font-shader-firmware-and-legacy-patterns.md` 补充字体、shader、side-channel、legacy/固件载体的识别信号和最小复现证据。
- 本轮不修改 raw 正文，不新增 wiki 页面，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-reverse-language-family-evidence-sections-20260706-201155.zip`。

## 2026-07-06 — Web 服务端 Family 证据段归一

- 为 `known-cves-and-n-day-exploits.md` 补充 CVE/N-day 入口的识别信号，明确组件版本、补丁边界、前置条件和后续 primitive 串联关系。
- 将 `parser-wrapper-and-legacy-ssrf-tricks.md`、`php-java-python-deserialization.md`、`xml-command-and-graphql-injection.md` 的“共同识别信号”归一为 `## 识别信号`，保留原有内容和 family 归属。
- 为 `path-traversal-ssrf-upload-and-rsc.md` 补充最小证据段，覆盖路径、SSRF、上传/解包和渲染器资源定位的可复现要求。
- 本轮不修改 raw 正文，不新增 wiki 页面，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-web-server-family-evidence-normalization-20260706-201508.zip`。

## 2026-07-06 — Family 识别信号标题归一

- 结构扫描发现大量旧 family 页已有“共同识别信号”内容，但标题未统一为 `## 识别信号`，导致核心段落校验误判为缺失。
- 将 reverse、pwn、crypto、forensics、misc、pentest、osint 等方向中已有的“共同识别信号”统一改为 `## 识别信号`，不改动正文语义和页面入链。
- `classical-xor-and-substitution-ciphers.md` 原本只有最小证据和变体路由，补充轻量古典/XOR/替换题的真实识别信号。
- 本轮不修改 raw 正文，不新增 wiki 页面，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-family-recognition-heading-normalization-20260706-201705.zip`。

## 2026-07-06 — Reverse Tooling 边界补强

- Tooling 页审校发现 `disassemblers-debuggers-and-basic-tools.md` 缺少明确触发证据，容易退化为工具清单；补充何时只用基础工具建立载体事实、何时转专门 tooling/technique。
- `frida-angr-lldb-and-x64dbg.md` 已有触发信号和失败状态，但缺少稳定调用方式；补充 Frida、angr、lldb/x64dbg 的调用前置条件和最小调用边界。
- `qiling-triton-pin-and-ldpreload.md` 补充重型模拟/插桩的触发证据和失败后转向，避免在普通断点或简单 patch 可解决时过早上重型工具。
- 本轮不修改 raw 正文，不新增 wiki 页面，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-reverse-tooling-boundary-clarity-20260706-204704.zip`。

## 2026-07-06 — Malware C2 Raw 模式补充

- 语义抽查 `raw/malware/c2-and-protocols.md` 时发现 Poison Ivy、DarkComet、Cobalt Strike 与 IRC C2 MITM 的可复用模式没有在 active technique 中明确承接。
- 在 `malware-c2-session-key-and-protocol-recovery.md` 增加已知 RAT 协议、Cobalt Strike Beacon 分支和“已知家族速查”表，补充触发证据、最短恢复路径和失败边界。
- 保留该页作为 technique，不新建单个 RAT 页面；这些模式共用“C2 协议/配置/会话恢复”主线，独立拆页会退化成工具或案例目录。
- 本轮不修改 raw 正文，不新增 wiki 页面，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-malware-c2-raw-pattern-ingest-20260706-204853.zip`。

## 2026-07-06 — AI/ML 与 Crypto 长尾 Raw 模式索引补强

- 抽查 `raw/ai-ml/adversarial-ml.md` 与 `wiki/adversarial-ml.md`，发现 raw 中 foolbox/Keras 梯度、data poisoning 和 backdoor detection 的可复用证据没有明确进入 family 案例索引。
- 在 `adversarial-ml.md` 补充 backdoor detection 路由和 raw 模式索引，明确 FGSM/PGD/C&W、patch、classifier evasion、poisoning/backdoor、foolbox 与手写 Keras gradient 的适用边界。
- 抽查 `raw/crypto/exotic-secret-sharing-rabin-and-polynomials.md`，为 `exotic-secret-sharing-rabin-and-polynomials.md` 补充资料内长尾模式索引，覆盖 Cayley-Purser、Asmuth-Bloom、Rabin polynomial primes、Vandermonde 和 LCG period。
- 本轮不修改 raw 正文，不新增 wiki 页面，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-ai-crypto-raw-pattern-index-20260706-205020.zip`。

## 2026-07-06 — Pwn 单入链 Raw 二级路由抽查

- 抽查只挂在 `pwn-first-pass-red-flags-and-protections.md` 的 raw，确认 `D3CTF2019-babyrop-wp`、`D3CTF2019-basic-basic-parser-wp`、`D3CTF2019-new-heap-wp`、`D3CTF2021-deterministic-heap-wp` 和 `D3CTF2021-easy-chrome-full-chain-wp` 不应只停留在首轮 family。
- `babyrop` 补入 VM/解释器与运行时 primitive 页面，保留“VM 指令写宿主返回地址”的 pivot；`basic-basic-parser` 补入 parser primitive 与 heap UAF 页面。
- `new_heap` 补入 heap 生命周期与 heap metadata/bin 页面，明确 glibc 2.29 tcache 检查、consolidation、stdout leak 和 hook 落点；`deterministic-heap` 补入 heap UAF 与 Windows 平台页，强调 NT Heap/LFH 稳定占位。
- `easy-chrome-full-chain` 补入 JIT/runtime 与 OOB/JIT primitive 页面，强调 V8 OOB primitive 到 Mojo sandbox escape 的衔接条件。
- 本轮不修改 raw 正文，不新增 wiki 页面，不调整 index 入口。
- 修复前备份：`D:/文档/markdown文件/ctf-wiki/backups/pre-pwn-single-ref-raw-routing-20260706-212154.zip`。
