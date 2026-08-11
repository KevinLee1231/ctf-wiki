# dungeon

## 题目简述

`dungeon` 把 63×63 迷宫打乱存为连续的 `state_t` 表；每个 24 字节 cell 含西、北、东、南四个 room 值及一个 action function 指针。room 值的 `0x10000000` 位表示门锁；进入带 action 的房间按 `p` 会执行函数，异或翻转一至三面墙及其对侧。目标是从启动房间到 flag 房间，必要时按一次正确按钮。

主要难点是从编译产物恢复被置换的迷宫表与各 action function 的墙翻转目标，再做图搜索，归入 Reverse。

## 解题过程

### 重建 state 表和按钮效果

官方 solver 按二进制布局读取 maze：文件偏移 `0x27080..0x3e498`，运行时虚拟地址 `0x428080..0x43f498`，每 24 字节解析为 4 个小端邻居值和 1 个函数指针。已知本构建的起点为 room 1251，终点为 room 1551。

对每个非空 action pointer，脚本用 Capstone 反汇编到 `ret`。函数里的 RIP-relative `lea` 指向某个 `MAZE[target]` 的四个 `int` 字段之一：相对 MAZE 起点除以 24 得到 target，余数除以 4 得到 `W/N/E/S`。这恰好就是按钮要异或 `LOCKED_BIT` 的门。

### 分两段 BFS

初始图仅把未置 `LOCKED_BIT` 的邻居作为边。对于每个按钮房间：

1. 在初始图 BFS 寻找起点到该房间的路径；
2. 深拷贝图，应用该 action 的所有 XOR 翻转；
3. 再 BFS 检查按钮房间能否到终点。

命中后拼接 `first_leg + 'p' + second_leg`，并将逻辑方向 `W/S/E/N` 映射为交互按键 `a/s/d/w`。无需尝试所有按钮状态组合，因为官方生成实例存在单个能解锁 flag 路径的 action。

### 验证

题目配置中的 flag 为 `DUCTF{you_make_it_out_of_the_dungeon_safely!}`。本文未登录 SSH 或执行生成的按键序列；迷宫数据布局、动作还原和 BFS 条件来自题目源码与官方 solver 的静态对照。

## 方法总结

- 核心技巧：大型状态表中包含函数指针时，函数对表的 RIP-relative引用常比盲目玩游戏更适合作为状态转换说明书。
- 识别信号：门锁位、按键触发的 XOR、置换的 room 编号和明示的 goal room，通常可以抽象成“动态边图”的最短路问题。
- 复用要点：先按原始状态搜索到开关，再复制并应用开关效应后搜索；不要在原图上累积修改，否则会污染后续候选按钮的判断。
