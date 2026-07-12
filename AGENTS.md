# AGENTS.md instructions for CTF Wiki

Always respond in Chinese-simplified.

# 角色与目标

你是 CTF Wiki 的维护者。这个 AGENTS.md 的核心作用是告诉智能体：这个 CTF 知识库应该怎样组织成可复用的技巧知识图谱。

本库不是按比赛、来源或文件类型堆放资料的目录，也不是 `ctf-writeup`、`pdf` 或专项 `ctf-*` skill 的规则副本。你的任务是把已经进入资料层的 writeup、博客、PDF 派生文本、工具文档、靶场复盘和本机经验，提炼成可查询、可迁移、可复用的 CTF 技巧图谱。

# 一、知识库分层

本库是 `ctf-*` skills 的外接补充知识源，不是每次解题都要加载的默认上下文。

```text
当前题目证据                    L0：题面、附件、服务、运行结果、已有分析
ctf-solve-challenge / SKILL.md  L1：全局分流、方向判断、何时进入专项 skill
ctf-* / SKILL.md                L2：专项边界、首轮判断、工具入口和 wiki 直链
ctf-wiki/wiki/*.md              L3：扁平技巧知识图谱
ctf-wiki/raw/<direction>/       L4：原始资料正文，作为证据和案例来源
```

目录职责：

```text
raw/       原始资料层；按正式方向归档；不承担知识图谱组织职责
wiki/      active 技巧知识图谱；所有页面直接位于 wiki/*.md
backups/   备份快照，只读；新增备份必须是 zip 压缩包，不保留展开目录
index.md   查询入口和页面索引
log.md     维护日志
AGENTS.md  维护规则
CLAUDE.md  指向 AGENTS.md
```

active tree 只保留 `raw/`、`wiki/`、`backups/` 和根文档。不要新增按比赛、年份、来源或题型继续嵌套的 active 知识目录。`backups/` 下的备份条目必须是 `.zip` 文件；若临时展开备份用于核对，核对完成后必须删除展开目录。

# 二、wiki 组织原则

`wiki/` 是本库主体，是面向未来解题和技巧复用的扁平知识图谱。页面不按比赛、不按来源、不按年份组织，而按可复用技术模式组织。wiki 文件名使用稳定英文 slug 或通用技术名；比赛名和题目名只作为案例来源，不作为技巧页主命名依据。一个 wiki 页面可以综合多个 raw 来源；一个 raw 也可以为多个 wiki 页面提供案例证据。

`wiki/` 中只使用三类页面：

- `family`：分流页、技巧族入口、首轮/方向页和跨技巧 map。负责连接多个 technique，并说明首轮判断、变体 pivot 和失败后转向。
- `technique`：具体可复用技巧、攻击模式、恢复路径或判断模型。负责适用场景、识别信号、最小证据、解法骨架和常见坑。
- `tooling`：工具入口、环境限制、稳定调用方式和常见失败状态。负责说明何时用工具、怎么稳定调用、失败后转向哪里。

不要在 `wiki/` 中引入第四类 active 页面。原先承担“首轮判断”“方向入口”“triage”“map”的页面统一视为 `family`。

wiki 页面通过 frontmatter 的 `raw:` 或正文“原始资料”引用来源。多个来源结论冲突时，不改写 raw，在 wiki 中写清适用边界和判断依据。不要把整篇 raw 搬进 wiki，只抽取能复用的模式、识别信号、证据和解法骨架。

wiki 页面轻量元数据建议：

```yaml
---
type: technique
tags: [crypto, web, jwt]
skills: [ctf-web, ctf-crypto]
raw:
  - ../raw/web/jwt-algorithm-confusion.md
updated: 2026-05-25
---
```

正文优先回答：

```markdown
# <技巧名称>

## 适用场景
## 识别信号
## 最小证据
## 解法骨架
## 关键变体
## 常见陷阱
## 关联技巧
## 原始资料
```

## technique 页

`technique` 页记录一个明确技巧、攻击模式、恢复路径或判断模型，例如 JWT algorithm confusion、Coppersmith small root、tcache poisoning、WASM VM lifting。

适合新建或扩展 technique 页的情况：

- 出现新的可复用技术模式；
- 出现更清晰的识别信号；
- 出现更短、更稳或更可迁移的解法骨架；
- 出现容易误判的 pivot；
- 出现能减少未来试错成本的最小证据判断。

不要为单题流水账、flag、比赛元数据、一次性路径或已有页面覆盖的常识新建 technique 页。

## family 页

`family` 是技巧族入口、首轮分流页或跨技巧 map，不是单纯分类目录；它和其它页面一样直接放在 `wiki/` 下。新建技巧族时优先用 `*-family.md` 命名；已经承担首轮判断、方向入口、triage 或 map 价值的稳定页面可保留语义化文件名，但 frontmatter 必须标为 `type: family`。

只有满足以下条件时才新建或保留 family 页：

1. 至少连接 3 个以上具体 technique 页；
2. 这些 technique 有共同识别信号；
3. 具体路线明显不同，需要先判断变体；
4. family 页能提供 pivot 价值，而不是只列链接。

family 页应回答共同适用场景、首轮变体判断、最小证据、失败后 pivot 和常见误判。若只剩目录作用，应合并回具体 technique 或删除。

首轮/方向 family 页可以引用 raw WP 作为 case routing 表；具体 technique 页也可以引用直接支撑该技巧的代表性 raw。不要强制每个 raw 同时挂到 family 和 technique；同一 raw 只有在确实支撑多个模式时才多处引用。

## tooling 页

`tooling` 页记录工具的使用边界、入口、环境限制、最小命令和常见错误。工具页不是安装流水账，也不是工具百科。

工具页应说明：适合什么问题、什么证据触发使用、当前环境如何稳定调用、常见失败状态、失败后转向哪里，以及它和对应 `ctf-* / SKILL.md` 的工具入口是否一致。

工具页必须写具体工具路由、环境边界和失败转向；不要使用“关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式”这类可套用到任意页面的占位句，也不要把 raw 标题机械展开成工具百科表。

## Query：按需查询

解题或回答问题时，不要一次性加载全库：

1. 先读已激活 `ctf-* / SKILL.md`。
2. 读取 skill 给出的 wiki 直链或 `<direction>-tooling.md`。
3. 信号不够明确时查 `index.md`。
4. 只读取最相关的 `wiki/*.md`。
5. 需要案例细节、命令输出、原始推导或资料出处时，再回查 `raw/<direction>/`。

# 三、raw 作为证据层

`raw/` 是原始资料层，不是规范化知识页，也不是 agent 随手改写的草稿区。它可以包含 WP、博客、PDF 派生文本、工具文档、复盘材料和其它用户导入资料，但本 AGENTS.md 不规定这些资料如何生成。

默认规则：

- 不默认改写 `raw/` 正文；
- 不把 raw 改造成 wiki 模板；
- 不给 raw 批量添加 frontmatter；
- 不为了“整齐”清洗 raw 的来源语义；
- 从 raw 提炼知识时，只读取和引用，抽象结果写入 `wiki/*.md`。

## raw 一级方向与归档判定

`raw/` 的正式一级方向固定为：

```text
web
pwn
reverse
crypto
forensics
stego
osint
malware
pentest
ai-ml
mobile
blockchain
hardware-embedded
cloud-infra
```

`raw/_unclassified/` 只用于暂时收纳无法归入安全技术方向的问卷、纯谜题、数据清洗或跨算法合集。它是待复核状态，不是第十五个正式方向；不得新建 `raw/misc/` 或 `raw/programming/` 作为兜底目录。

归档只选择一个物理主目录，判定依据是“最小预期解法中的决定性主障碍”，而不是赛事原始标签、附件扩展名、部署位置或使用过的工具。目标平台只有在其独有安全模型决定解法时才优先成为主方向：

| 方向 | 进入条件 |
|---|---|
| `web` | HTTP/API、认证授权、会话、浏览器或服务端应用逻辑是主障碍。 |
| `pwn` | 需要构造程序、内核、运行时或沙箱利用原语，突破执行或权限边界。 |
| `reverse` | 静态/动态分析程序、字节码、VM 或固件并恢复行为本身就是终点。 |
| `crypto` | 密码算法、协议、随机数、数论结构、密码实现缺陷，或 Base/hex/URL/ROT、字符编码、自定义码表和多层可逆编码等表示层变换决定解法。 |
| `forensics` | 从磁盘、内存、日志、PCAP、文件系统或其它数字证据恢复并关联事实。 |
| `stego` | 核心是定位、重组并提取被隐藏、嵌入或碎片化的信息，包括隐蔽信道、QR 碎片、视觉/空间线索和游戏场景中的隐藏信息。 |
| `osint` | 主要证据来自外部公开信息源，需要搜集、关联和验证。 |
| `malware` | 恶意行为、持久化、配置、C2、恶意协议或载荷链是主障碍。 |
| `pentest` | 凭据、服务枚举、横向移动、隧道、提权或多阶段主机/网络攻击链占主导。 |
| `ai-ml` | 模型行为、对抗样本、模型提取、成员推断、投毒或 LLM/Agent 攻击决定解法。 |
| `mobile` | Android/iOS 组件、IPC、签名、权限、Keystore/Keychain 或平台运行时机制决定解法。 |
| `blockchain` | 智能合约、交易、链上状态、合约 VM、存储布局或链特有权限语义决定解法。 |
| `hardware-embedded` | JTAG/UART、总线、MCU、RF、侧信道、Secure Boot 或硬件/嵌入式机制决定解法。 |
| `cloud-infra` | 云 IAM、控制面、资源策略、对象存储、Serverless、编排、IaC、CI/CD 或供应链决定解法。 |

高频边界按以下规则处理：

- 理解程序行为即完成目标归 `reverse`；理解后还要构造利用突破执行/权限边界归 `pwn`。
- 数字证据恢复归 `forensics`；隐藏或碎片化载荷、QR 重组、视觉/空间线索及游戏场景隐藏信息的提取归 `stego`；外部公开来源调查归 `osint`。
- Base/hex/URL/ROT、字符编码、自定义码表和多层可逆编码等普通表示层编码归 `crypto`；编程语言、脚本编写和约束求解只是手段，仍按实际决定解法的主障碍归类，不能默认归 `crypto`。
- 普通 APK/IPA/Native 算法还原仍归 `reverse`；只有移动平台安全模型本身是主障碍时才归 `mobile`。
- 普通 dApp 前后端漏洞归 `web`，密码数学缺陷归 `crypto`；链上状态、EVM/合约语义主导时才归 `blockchain`。
- 普通固件反汇编归 `reverse`，固件内存利用归 `pwn`；物理接口、总线、Secure Boot、RF 或侧信道主导时才归 `hardware-embedded`。
- 部署在云上的普通 Web 漏洞仍归 `web`；IAM、资源图和云控制面主导时才归 `cloud-infra`。
- PCAP 事件重建归 `forensics`，恶意 C2/配置恢复归 `malware`，完整网络/域攻击链归 `pentest`。

`network`、`firmware`、`iot`、`container`、`kubernetes`、`jail`、`sandbox`、`fullpwn`、`attack-defense`、比赛名、年份和来源都不是 raw 正式一级方向；它们应由文件名、wiki 关联、正文语义或后续元数据表达。移动 raw 时必须连同同 basename 资源目录一起移动，并同步更新 `wiki/*.md` 中的 raw 引用；除路径确有必要外，不改写 raw 正文。

只有用户明确要求整理、迁移、删除、替换、重命名 raw，或对应专门 skill 在本轮任务中按用户要求生成/替换资料时，才修改 raw。若 raw 与 wiki 内容冲突，优先在 wiki 中写清冲突和适用边界，不要直接改写 raw 来消除冲突。

# 四、禁止事项

- 不把 AGENTS.md 写成其它 skill 的规则副本。
- 不在 `wiki/` 下按比赛、年份、来源、题目继续分目录。
- 不新建只有目录作用、没有 pivot 价值的 family 页。
- 不新建 `misc.md`、`advanced.md`、`notes.md`、`tricks.md` 这类弱语义页面。
- 不把单题流水账直接搬进 wiki 主线。
- 不为了整理 wiki 顺手改写 raw 正文。
- 不让新增页面只靠文件名存在；必须有查询入口或相邻页面入链。
- 不把工具安装流水账当作 tooling 页主体。

# 五、沉淀与审校流程

维护流程分为三类：`Ingest` 负责把资料提炼进 wiki；`Structural Lint` 负责确认结构和链接没有被破坏；`Semantic Health Check` 负责在用户明确要求时复核 raw、technique、family 和 index 之间的语义关系。

## Ingest：沉淀知识

当用户在 `raw/` 中加入新资料，或明确要求整理知识库时，先判断资料是否提供复用价值。

如果资料没有提供新的识别信号、解法骨架、pivot 或工具边界，只保留在 raw，不进入 wiki。

值得沉淀的内容：

- 新的可复用技术模式；
- 清晰识别信号；
- 可迁移解法骨架；
- 容易误判的 pivot；
- 工具链经验、环境限制或稳定调用方式；
- 多个 raw 指向同一模式，值得抽象成统一 technique 或 family。

不应沉淀的内容：

- 单题完整流水账；
- flag、剧情、比赛元数据、一次性路径；
- 已有 wiki 覆盖且没有新增识别特征或边界的常识；
- 会污染 wiki 主线的细节；
- 个人复盘或完整思维过程。

写入位置按以下规则判断：

- 具体技巧和案例模式写入 `wiki/*.md`；
- 工具、路径、安装状态、调用层差异写入 `wiki/<direction>-tooling.md`；
- 没有合适页面时，新建语义化文件名，不使用 `misc.md`、`advanced.md`、`notes.md`、`tricks.md` 这类弱名称。

新增或显著改动的 wiki 页必须有入链：来自 `index.md`、family 页、相邻 technique 页或对应 skill。

执行顺序：

1. 判断资料是否有可复用技巧、识别信号、工具经验或 pivot 价值。
2. 查找已有 `wiki/*.md`，能合并就补充已有页面。
3. 只有确有新技术模式、新分流价值或新工具边界时才新建页面。
4. 新增或显著改动 wiki 页时建立入链。
5. 判断是否影响对应 skill；如影响，直接同步修改。
6. 更新 `index.md` 和 `log.md`。
7. 运行结构和链接校验。

skill 协同规则：

- `ctf-wiki` 不复制 `ctf-writeup`、`pdf` 或专项 `ctf-*` skill 的规则。
- 新知识影响首轮判断、常用 pivot、工具路径或高频页面直链时，同步修改对应 `ctf-* / SKILL.md`。
- 需要同步 skill 时直接修改对应 `SKILL.md`，并在 `log.md` 记录原因。
- `ctf-solve-challenge` 是分流入口；它的 `references/` 是本地速查资料，不替代本库 wiki 图谱。

## Structural Lint：结构和一致性审校

结构性修改后至少检查：

- `raw/` 是否只使用 14 个正式方向和可选的 `_unclassified/` 暂存区，且没有重新出现 `misc/` 或 `programming/`；
- `_unclassified/` 是否仍只承担待复核状态，并在 `index.md` / `log.md` 中记录数量和保留原因；
- raw Markdown 与同 basename 资源目录是否位于同一方向，wiki 中指向被迁移 raw 的引用是否同步更新；
- `wiki/` 是否仍为扁平图谱；
- `wiki/*.md` 是否只使用 `family`、`technique`、`tooling` 三种 `type`；
- Markdown 链接是否存在断链；
- `index.md` 是否覆盖所有 active wiki 页面；
- 新增页是否有来自 index、family、相邻 technique 或 skill 的入口；
- family 页是否仍提供 pivot 价值；
- tooling 页与 `ctf-* / SKILL.md` 是否一致；
- 页面过长且出现两个以上独立技巧时，是否应拆分；
- 重复页面是否应合并，并保留更语义化的文件名；
- 是否误改了 `raw/` 正文；
- `index.md` 和 `log.md` 是否按修改范围更新。

## Semantic Health Check：语义健康检查

只有当用户明确要求“健康检查”“重新组织知识网络”“检查 raw 概括准确性”“检查 technique/family 结构”或类似任务时，才执行语义健康检查。不要把它作为每次普通结构性修改后的全量强制步骤。

语义健康检查的推荐顺序：

1. 从 `technique` 页开始逐篇抽查或全量检查：判断页面引用的 raw 是否真的支撑该技巧；页面是否过宽、过窄、命名不准确、缺少最小证据、缺少关键 pivot，或是否与其它 technique 重复。
2. 判断 technique 是否应合并、拆分、重命名或删除：
   - 合并：识别信号、最小证据、解法骨架和失败 pivot 基本相同，只是来源或命名不同。
   - 保留拆分：第一步操作、工具链、证据形态、运行时语义或 skill 边界不同。
   - 删除或并入：没有独立识别信号、没有独立 raw 支撑、没有入链，或只重复 family 内容。
   - 拆分：一个 technique 页中出现两个以上互不依赖的攻击模式，且它们的首轮判断不同。
3. 根据 technique 调整 family 页：确认下一跳是否准确，变体计数是否合理，family 是否仍提供 pivot 价值，而不是只列链接。
4. 最后更新 `index.md`：保持分类索引职责，只更新页面入口和分类归属，不把细粒度判断重新搬回 index。
5. 更新 `log.md` 记录语义调整原因、合并/删除/保留的取舍和校验结果；必要时同步对应 `ctf-* / SKILL.md`。

# 六、index / log 协同

`index.md` 管“怎么找到页面”，定位为轻量分类索引，不替代 family 页的分流判断，也不替代 technique 页的细节内容。

- `index.md` 应按页面类型和方向组织入口：family、tooling、按方向分类的 technique、raw 统计和维护入口。
- 题面信号、失败状态和下一跳 pivot 应主要写在 family 页或具体 technique 页中，避免 index 退化成长速查表。
- 新增、合并、删除或显著重写 wiki 页面时更新 `index.md`；raw-only 变动只有影响查询入口时才更新。

`log.md` 管“为什么这么改”，记录结构性维护动作和重要取舍。

- 新增、合并、删除或显著重写 wiki 页面时更新 `log.md`。
- 调整 family、index 入口、skill 直链，或批量迁移、删除、重命名 raw / active tree 时更新 `log.md`。
- 明确决定某批 raw 暂不沉淀到 wiki 时，可以在 `log.md` 记录原因。

# 七、维护要求

每次结构性修改后必须确认：

- active tree 保持为 `raw/`、`wiki/`、`backups/` 和根文档；
- `raw/` 的正式目录保持为 14 个方向；如存在 `_unclassified/`，它只作为待复核暂存区；
- `raw/` 中不存在重新建立的 `misc/`、`programming/`，也不存在因迁移遗留的孤立资源目录；
- `wiki/` 下只包含 Markdown 页面；
- `wiki/*.md` 只使用 `family`、`technique`、`tooling` 三种页面类型；
- `backups/` 下新增和保留的备份均为 `.zip` 压缩包，不遗留展开备份目录；
- active Markdown 的知识链接和 raw 链接未断；
- `index.md` 和 `log.md` 已按本轮修改范围更新；
- 除明确授权的 raw 整理、结构迁移或资料替换外，不应改动已有 raw 正文；
- family 页必须仍有明确 pivot 价值。

# 八、本地提交规则

当智能体在本轮任务中新增、修改、删除或重命名 `wiki/*.md` 时，应在一个逻辑完整的维护单元完成并通过校验后，自动创建一次本地 Git 提交，无需用户再次确认。不要把一次任务中的逐页保存机械拆成多个碎片提交；多个互不相关的维护目标则应分别提交。

提交前必须满足：

- 已完成相关 `index.md`、`log.md`、相邻 wiki 页面和 raw 引用的必要同步；若本轮判断同时影响对应 skill，还应按本文件的 skill 协同规则完成外部同步，但仓库外的 skill 改动不属于本仓库提交范围；
- 已运行与本轮修改范围相符的结构和链接校验，且不存在断链、非法页面类型、index 漏项或其它已知结构错误；
- 已审阅准备提交的完整差异，确认内容完整、可理解且不包含敏感信息、临时文件或明显未完成内容；
- 提交信息统一使用简洁、明确的中文，准确说明修改对象和目的，不使用“更新文件”“修改内容”“调整一下”等无意义描述。

既有改动处理规则：

- 任务开始前已经存在的改动不应一概排除。只要经审阅确认与当前知识库维护目标相关、内容完整且能够通过校验，即使无法确认来自用户还是此前的智能体，也可以纳入同一次本地提交；
- 与当前目标无关、明显未完成、包含敏感信息或无法验证正确性的既有改动，不得直接混入；应保留在工作区，或在确有保留价值时单独创建说明明确的本地检查点提交；
- 不得在未审阅差异的情况下机械执行 `git add -A` 并提交全部内容。

原子性要求：

- 同一维护目标下的 wiki 页面及其配套 `index.md`、`log.md` 应放入同一提交；
- 若 wiki 调整依赖 raw 的新增、迁移、重命名或引用更新，必须将必要的跨层改动放入同一原子提交，不得为了按目录拆分而制造断链的中间状态；
- 本仓库的暂存和提交范围只能包含仓库根目录下的文件；位于仓库外部的对应 skill 修改不得纳入本仓库提交；
- 若本轮同时同步了仓库外 skill，应将其作为独立的外部协同结果处理，并在交付说明中单独列出路径和修改目的；跨仓库一致性通过校验和记录保证，不得声称它与本仓库改动属于同一原子提交。

以下情况禁止自动提交：

- 校验失败或任务尚未完成；
- 准备提交的改动中仍有无法判断用途或正确性的内容；
- 用户明确要求不要提交；
- 提交必须包含超出当前任务范围且未经审阅的改动。

自动提交只限本地，不得自动执行 `git push`。提交完成后必须向用户报告提交哈希、提交说明和包含范围。
