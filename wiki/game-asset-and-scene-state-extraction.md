---
type: technique
tags: [reverse, stego, game, asset, scene, state]
skills: [ctf-reverse, ctf-stego]
raw:
  - ../raw/web/game-state-websocket-and-wasm.md
  - ../raw/reverse/android-games-hardware-and-runtime-platforms.md
  - ../raw/stego/SUCTF2026-Artifact_OnlineWP.md
updated: 2026-07-27
---

# Game Asset and Scene-State Extraction

## 适用场景

Flag 或关键状态位于游戏资源包、场景层级、脚本数据、存档、隐藏对象或客户端/服务端同步字段中；先恢复数据模型，再判断是修改状态还是提取隐藏载荷。

## 识别信号

- Unity/Unreal/Godot/Electron/Web 游戏资源、scene、bundle、save 或 websocket 状态。
- 画面不可见对象、层级外坐标、禁用组件或资源 metadata 含线索。
- 客户端显示与服务端 authoritative state 不一致。

## 最小证据

- 确认引擎/容器版本和资源索引。
- 将对象名、组件、坐标、脚本字段与运行时行为对应。
- 状态修改需证明服务端接受；隐藏信息需可从资源/场景独立恢复。

## 解法骨架

1. 枚举 bundle/scene/script/save 和网络消息 schema。
2. 提取资源与对象层级，搜索禁用、隐藏、越界和动态生成对象。
3. 对客户端状态与网络包做差分，识别权威校验位置。
4. 选择资源提取、存档修改、协议重放或运行时内存修改并复验。

## 关键变体

- Scene/object hidden clue。
- Asset bundle/script decompilation。
- WebSocket/client-state manipulation。

## 常见陷阱

- 只用截图寻找线索，忽略场景树和资源 metadata。
- 修改客户端 UI 但服务端状态未变化。
- 把普通游戏资源误归 stego，未证明隐藏位置决定解法。

## 关联技巧

- [game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md)
- [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)
- [custom-vm-and-wasm-state-lifting.md](custom-vm-and-wasm-state-lifting.md)

## 原始资料

- [game-state-websocket-and-wasm.md](../raw/web/game-state-websocket-and-wasm.md)
- [android-games-hardware-and-runtime-platforms.md](../raw/reverse/android-games-hardware-and-runtime-platforms.md)
- [SUCTF2026-Artifact_OnlineWP](../raw/stego/SUCTF2026-Artifact_OnlineWP.md)
