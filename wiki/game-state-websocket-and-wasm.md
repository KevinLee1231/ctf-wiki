---
type: family
tags: [misc, family, game-state, websocket, wasm]
skills: [ctf-misc, ctf-web, ctf-reverse]
raw:
  - ../raw/misc/game-state-websocket-and-wasm.md
updated: 2026-06-12
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
| server time only check | 校验是否只依赖客户端提交时间或可预测时间 | 枚举时间窗口或跳过等待 |
| de Bruijn / coverage | 胜利条件是覆盖所有子串或状态组合 | 构造序列，减少交互次数 |
| Brainfuck / esolang interpreter | 需要插桩、跟踪 tape 或修改解释器行为 | 转 [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) 或 Reverse |
| WASM linear memory | flag、坐标、生命值或 check buffer 在 memory 中 | Patch memory/函数或重放导出调用 |
| Minecraft/RPG/map resource | 事件位置、插件逻辑或资源包隐藏路径 | 解析资源，脚本化移动或状态触发 |

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
