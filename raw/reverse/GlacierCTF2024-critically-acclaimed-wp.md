# GlacierCTF 2024 Critically Acclaimed

## 题目简述

题目把 Final Fantasy XIV 的八人副本机制做成了一个 32×32 网格生存游戏。服务端每回合解析一行最多 10 个字符：`u/d/l/r` 移动，`a` 攻击，`h` 治疗，`b` 格挡；随后结算该 tick 的 AOE、甜甜圈、激光、分摊塔、击退、凝视、全屏伤害等机制。玩家有 100 点生命，boss 有 32 点生命，击败 boss 后服务端输出 flag。

官方客户端没有完成网络交互，官方 WP 也明确提示直接阅读 JSON。仓库同时给出 `mechanics.yml`、`player_movement.json`、Rust 服务端和官方路线生成器，因此可以离线还原每回合的安全位置、动作与朝向，再把整条路线一次发送给服务。

## 解题过程

### 1. 还原一回合的输入与结算顺序

服务端只处理每行前 10 个字符。移动字符可以连续执行；遇到 `a/h/b` 后立即停止解释本行，然后调用 `game.advance()` 结算机制：

```rust
for c in player_input.chars().take(10) {
    match c {
        'u' => player.player_move(Direction::N),
        'd' => player.player_move(Direction::S),
        'l' => player.player_move(Direction::W),
        'r' => player.player_move(Direction::E),
        'a' => { damage_boss = true; break; }
        'h' => { player.heal(); break; }
        'b' => { player.block(); break; }
        _ => (),
    }
}
game.advance()?;
if damage_boss { game.damage_boss(); }
```

因此一行就是“先走到本回合落点，再选择攻击、治疗或格挡”。攻击在机制结算后才扣 boss 一点血；玩家必须活着通过结算，才能累计这次攻击。

玩家初始位置为 $(15,15)$。移动会记录 `moved` 和最后方向；每 tick 结束后才清零。Pyretic 在本回合发生移动时造成 70 点伤害，而 Gaze 根据最后方向与视线方向判断玩家是否面向凝视源。格挡将下一次正伤害减半，治疗恢复 80 点且生命上限为 100。

### 2. 使用官方安全轨迹

`player_movement.json` 的每项格式为：

```text
[[目标 x, 目标 y], 动作, 朝向修正]
```

动作 `a/h/b/n` 分别表示攻击、治疗、格挡、不执行额外动作；朝向编号 `1/2/3/4` 分别要求最终方向为上、下、左、右。路线已经避开 `mechanics.yml` 中 0–49 tick 的全部危险区域，解题只需把相邻目标点转成移动字符串。

路线生成器的核心逻辑为：

```python
import json

moves = json.load(open("player_movement.json"))
x, y = 15, 15

for i, (pos, action, orientation) in enumerate(moves):
    sx = "l" * max(0, x - pos[0]) + "r" * max(0, pos[0] - x)
    sy = "d" * max(0, y - pos[1]) + "u" * max(0, pos[1] - y)
    line = sx + sy

    # 用往返一步改变最后朝向，同时回到同一格。
    if orientation == 1 and not line.endswith("u"):
        line += "du"
    elif orientation == 2 and not line.endswith("d"):
        line += "ud"
    elif orientation == 3 and not line.endswith("l"):
        line += "rl"
    elif orientation == 4 and not line.endswith("r"):
        line += "lr"

    if action != "n":
        line += action
    print(line)

    x, y = pos
    if i == 4:
        x, y = 29, 15
    if i == 48:
        x, y = 29, 25
```

两个位置修正来自 tick 4 和 tick 48 的 `Wind` 强制击退。`player.force_move_by()` 不会设置主动移动标志，也不会更新最后方向，所以生成器必须单独更新坐标，不能把它误算成普通移动。

### 3. 解释几个容易误判的机制

- AOE、Donut、Laser、Cleave 和持续 Puddle 按回合结束时的最终坐标判定；经过危险格本身不会受伤。
- Tower 伤害会随离塔距离增加，路线需要靠近塔；Flare 则应尽量远离。
- Gaze 并非单纯看坐标，还检查最后一次主动移动方向。若已到安全点但方向错误，可追加 `du`、`ud`、`rl` 或 `lr` 原地往返。
- Pyretic 只看本 tick 是否主动移动。预定路线在相关回合保持不动，必要时仅治疗或格挡。
- Raidwide 与其他机制可能同 tick 叠加，路线用 `h` 和 `b` 管理生命值。

官方 `sol.txt` 共 50 行；直接逐行发送即可。第 32 次 `a` 令 boss 血量归零，服务端返回：

```text
gctf{Th3yre_d0Dg1ng_3EV3RY_att4ck!_4150d41b62fe9b9d}
```

## 方法总结

本题不需要猜迷宫或手工试玩全部机制。先从服务端确定“输入解析 → 机制结算 → boss 扣血”的顺序，再把公开 JSON 中的目标位置转成逐回合命令即可。最关键的实现细节是强制击退不会更新 `moved/last_movement`，而凝视和 Pyretic 恰好依赖这两个状态；路线生成时应把坐标、最后朝向和主动移动标志分别建模。
