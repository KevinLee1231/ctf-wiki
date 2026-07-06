---
type: family
tags: [misc, family, game-state, websocket, wasm]
skills: [ctf-misc, ctf-web, ctf-reverse]
raw:
  - ../raw/misc/game-state-websocket-and-wasm.md
  - ../raw/misc/SU_Artifact_OnlineWP.md
  - ../raw/misc/ACTF2026-master-of-album-wp.md
  - ../raw/misc/VNCTF2026-minecraft-wp.md
updated: 2026-07-06
---

# Game State, WebSocket and WASM

## 作用边界

本页是游戏状态、WebSocket 交互、客户端资源、WASM 线性内存和轻量解释器题的二级 family。它从 [vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md) 进入，负责判断是改客户端状态、重放协议、解 session、patch WASM，还是把规则转成约束/模拟。

如果主要工作已经变成复杂二进制逆向，转 Reverse；如果已经出现内存破坏 primitive，转 Pwn；如果是纯 Web 业务参数，转 Web 首轮页。

## 识别信号

- 题目表现为小游戏、地图、资源包、RPG/Minecraft、checkpoint、分数、时间限制、WebSocket 帧或客户端状态。
- Cookie/session/localStorage 中保存游戏状态，或 Flask session 暴露签名/明文结构。
- WASM 导出函数、linear memory、JS glue code 或浏览器 DevTools 可直接观察胜利条件。
- 需要覆盖所有子串、路径、状态或指令，例如 de Bruijn、Brainfuck、迷宫、地图事件。

## 最小证据

- 状态字段、胜利条件和服务端校验点：分数、坐标、时间、背包、checkpoint、签名 cookie。
- 可重放交互：HTTP 请求、WebSocket 帧、游戏资源变更或 WASM 函数调用。
- 客户端与服务端各校验什么；只改前端 UI 不算成功。
- 对 WASM/解释器，先定位 memory offset、导出函数、输入格式和 flag 检查位置。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| Cookie checkpoint / game state | 状态是否签名、加密、可枚举或只在客户端校验 | 改 cookie/session 或脚本化状态推进 |
| Flask session | secret 是否弱、是否可解码、字段是否被服务端信任 | 转 [auth-jwt.md](auth-jwt.md) 或 token family |
| WebSocket game | 帧格式、动作序列和服务器响应是否可重放 | 写 replay client，固定状态和时间 |
| Socket.IO quiz / media recognition | 题型、正确率阈值、session 校验和题库/媒体识别接口 | 写自动答题 client，分离封面、音频和题型分支 |
| server time only check | 校验是否只依赖客户端提交时间或可预测时间 | 枚举时间窗口或跳过等待 |
| de Bruijn / coverage | 胜利条件是覆盖所有子串或状态组合 | 构造序列，减少交互次数 |
| Brainfuck / esolang interpreter | 需要插桩、跟踪 tape 或修改解释器行为 | 转 [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) 或 Reverse |
| WASM linear memory | flag、坐标、生命值或 check buffer 在 memory 中 | Patch memory/函数或重放导出调用 |
| Minecraft/RPG/map resource | 事件位置、插件逻辑或资源包隐藏路径 | 解析资源，脚本化移动或状态触发 |
| Cube/面板状态选择器 | 字符映射、状态置换、激活规则和服务端执行点 | 建模状态转移，用 BFS/beam search 生成动作序列 |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [SU_Artifact_OnlineWP](../raw/misc/SU_Artifact_OnlineWP.md) | `5x5` 六面 cube 的 `R/C/F` 置换和 activate 选字符规则可本地模拟；先从符文文本恢复映射，再搜索能执行 `cd ..;nl flag` 的状态。 |
| [ACTF2026-master-of-album-wp](../raw/misc/ACTF2026-master-of-album-wp.md) | Socket.IO 问答系统没有强绑定登录态，随机 `team/token/session_id` 也能开局；重点是封面/音频识别缓存、题型分支和失败轮次丢弃策略。 |
| [VNCTF2026-minecraft-wp](../raw/misc/VNCTF2026-minecraft-wp.md) | Minecraft 题先画代理、子服、插件和共享数据库边界；creative 侧游戏权限只是入口，后续需转 Pentest/CVE 页面处理跨服权限和 Log4Shell。 |
| [D3CTF2023-d3craft-wp](../raw/misc/D3CTF2023-d3craft-wp.md) | Minecraft/PaperMC 移动事件和数据包语义差异可绕过，先逆插件目标位置再脚本化状态移动。 |
| [D3CTF2025-d3rpg-signin-wp](../raw/misc/D3CTF2025-d3rpg-signin-wp.md) | RPG 地图、标牌、隐藏路径和摩斯地板是主线，先复现游戏状态和资源线索。 |
| [NCTF2026-鸡爪流高手-wp](../raw/reverse/NCTF2026-鸡爪流高手-wp.md) | 游戏协议和 ELO 结算是主线；低分保护检查更新前状态，结算写入无符号分数字段导致下溢登榜。 |
| [NCTF2026-pay-for-2048-wp](../raw/reverse/NCTF2026-pay-for-2048-wp.md) | Electron `app.asar` 中 JS bridge 调 WASM license 校验；先直接用 Node 调应用服务，再补 WASM digest/key 派生。 |
| [SU_sqliWP](../raw/web/SU_sqliWP.md) | `/api/query` 注入前必须复现 Go WASM 前端签名和浏览器指纹字段；签名过后用 PostgreSQL 字符串拼接构造布尔盲注。 |

## 合并与拆分结论

- 保留为 family：raw 横跨 WebSocket、session、WASM、游戏资源和解释器，首轮 pivot 比单点 payload 更重要。
- 不合并进 `vm-z3-sandbox-and-game-basics.md`：总页只决定是否进入规则系统，本页负责游戏/客户端状态内部路线。
- 暂不拆 WASM、WebSocket 或游戏 session 小页：当前 raw 多为短案例，拆分后会形成弱入口。

## 常见误判

- 只改前端显示，没有触发服务端胜利条件。
- WebSocket 只抓一帧，没有重放握手、session 和动作顺序。
- WASM 只看反编译伪代码，没检查 JS glue 和 linear memory 的真实布局。
- 游戏题没有保存地图、资源包和事件脚本，导致状态不可复现。

## 关联页面

- [vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md)
- [misc-cross-category-triage-family.md](misc-cross-category-triage-family.md)
- [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md)
- [auth-jwt.md](auth-jwt.md)
- [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)
- [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md)

## 原始资料

- [game-state-websocket-and-wasm.md](../raw/misc/game-state-websocket-and-wasm.md)
- [SU_Artifact_OnlineWP.md](../raw/misc/SU_Artifact_OnlineWP.md)
- [ACTF2026-master-of-album-wp.md](../raw/misc/ACTF2026-master-of-album-wp.md)
- [VNCTF2026-minecraft-wp.md](../raw/misc/VNCTF2026-minecraft-wp.md)
