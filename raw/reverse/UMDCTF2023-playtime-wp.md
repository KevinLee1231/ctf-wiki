# Playtime

## 题目简述

题目提供一个修改过的 Pokémon Red Game Boy ROM，并说明 flag 藏在游戏中。普通游玩很难触达目标区域；逆向 ROM 的地图表可以发现一个额外加入、没有正常入口的隐藏地图。

## 解题过程

Pokémon Red 的地图由地图编号、头指针表、头部、对象表和块数据共同描述。对 ROM 字符串和地图表做差异检查，可发现额外的地图编号：

```asm
map_const UMDCTF_FLAG, 47, 5 ; $69
```

对应头部使用 `OVERWORLD` 图块集，尺寸为 $47\times5$ 个 metatile：

```asm
map_header UMDCTFFlag, UMDCTF_FLAG, OVERWORLD, 0
```

对象表没有 NPC、背景事件或普通 warp：

```asm
def_warp_events
def_bg_events
def_object_events
```

这解释了为什么正常流程无法进入。`UMDCTFFlag.blk` 恰好有
$47\times5=235$ 字节，每个字节选择一个地图块。可在支持 Game Boy 调试的模拟器中把当前地图编号改为 `0x69`，或把任一已有出口的目标地图临时补丁为 `0x69` 后重新进门。加载隐藏地图后，地形块在屏幕上拼出文字：

```text
HOPEUHADFUN
```

按标准格式提交：

```text
UMDCTF{HOPEUHADFUN}
```

如果不运行 ROM，也可以用 Pokémon Red 的 `OVERWORLD` metatile 表离线渲染
`UMDCTFFlag.blk`；关键是按地图宽度 47 分行，不能把 235 字节当连续字符读取。

## 方法总结

- 核心技巧：从地图头指针表和异常地图常量定位隐藏内容，而不是盲目通关整部游戏。
- “地图存在但没有 warp”是典型不可达资源信号，可通过调试器改地图 ID 或补丁已有入口访问。
- Game Boy 地图块编号不是 ASCII；必须结合图块集与地图宽高渲染，才能看到地形文字。
