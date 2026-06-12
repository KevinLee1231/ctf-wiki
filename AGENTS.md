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
raw/       原始资料层；按方向归档；不承担知识图谱组织职责
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
- `wiki/` 下只包含 Markdown 页面；
- `wiki/*.md` 只使用 `family`、`technique`、`tooling` 三种页面类型；
- `backups/` 下新增和保留的备份均为 `.zip` 压缩包，不遗留展开备份目录；
- active Markdown 的知识链接和 raw 链接未断；
- `index.md` 和 `log.md` 已按本轮修改范围更新；
- 除明确授权的 raw 整理、结构迁移或资料替换外，不应改动已有 raw 正文；
- family 页必须仍有明确 pivot 价值。
