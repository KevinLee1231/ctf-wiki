# godot

## 题目简述

题目是 Godot 游戏。发布版资源经过引擎加密；官方说明指出，先从运行进程内存取得项目 AES key，才能用 Godot 资源工具查看脚本。解出资源后，真正决定结局的不是贴图或对白，而是 `player.gd` 与 `shop.gd` 的两个布尔状态 `lucky`、`godot`。

源码项目可见角色、场景和大量装饰性素材，但这些 PNG/tileset 不携带不可替代的解题信息。WP 只保留 GDScript 状态转移，不复制游戏画面或素材目录。

## 解题过程

### 解开资源并定位状态机

对只有发布包的手册，先在运行游戏时取得 Godot 的 AES key，再用该 key 解出项目资源；官方说明将这一步明确为主要障碍。解出后从 `project.godot` 的主场景进入 `player.gd`：在商店区域按交互键时，只有同时满足 `lucky && godot` 才会执行隐藏传送。

```gdscript
if shop:
    if lucky and godot:
        global_position.x = 0
        global_position.y = -10000
```

这说明所谓通关条件只是两个状态，而不是对对白文本或地图图片做隐写分析。

### 取得 `lucky` 与 `godot`

`area_2d.gd` 在玩家进入 Pozzo 的 Area2D 时直接设置 `body.lucky = true`。正常游戏中到达 Pozzo 即可取得该状态。

`shop.gd` 的时间逻辑更隐蔽：初始化时记录 `check = now - 86400`；只要运行中的系统时间恰好等于这个记录值，就显示 Godot 并设置商店状态。按官方方法，启动游戏后把系统时间调到约一天之前并等待时间经过该秒，使 `now == check` 成立；随后重新进入商店触发区，使 `_on_body_entered` 把 `shop.godot` 复制到 `player.godot`。源码审阅时也可直接把两个 player flag 设为 `true` 后在调试项目中验证状态机。

最后携带两个 true 状态在商店交互，角色被传送到 $y=-10000$ 的隐藏区域，游戏输出 flag。

### 验证

官方题解给出的路径是“取得 AES key → 解出项目 → 激活 Lucky → 让 Godot 出现”；`player.gd` 精确证实两个布尔值共同触发传送。本文未启动游戏、抓取内存或运行 Godot，仅依据官方说明和已给出的 GDScript 交叉核对。

## 方法总结

- 核心技巧：游戏逆向优先从场景脚本恢复状态机，区分资源加密的入口障碍和最终的逻辑条件。
- 识别信号：Godot 发布包需要 key 才能导出资源时，优先寻找运行时 key；资源解出后用 `flag`、`global_position`、输入回调和场景触发器搜索结局分支。
- 复用要点：大批素材通常只是渲染资产；只有脚本无法表达的视觉线索才应保存截图，本题的触发条件已被源码完整表达。
