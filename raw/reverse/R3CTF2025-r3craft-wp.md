# r3craft

## 题目简述

附件是一套 Minecraft 1.21.6 Paper 服务端，加载了 GrimAC 反作弊插件和自定义 `R3Craft.jar`。玩家出生在一个由磨制安山岩围起的平台中，普通跳跃无法越过高约 1.5 格的墙。

反编译 `R3Craft.jar` 后可知，执行 `/flag` 时必须同时满足：

1. 登录位置的方块高度不高于 `Y_PLATFORM = -60`；
2. 发命令时玩家的 $Y$ 坐标至少为 `Y_BORDER + 2.2252 = -56.7748`；
3. 10 tick 后仍在线、未被判定已踢出，并且 `blockY > Y_BORDER = -59`。

因此题目不是寻找地图中的隐藏物品，而是研究客户端移动、挑战插件与 GrimAC 预测模型之间的边界。

## 解题过程

### 确认反作弊事件存在一次容错

挑战插件订阅 GrimAC 的 `FlagEvent`，计数代码等价于：

```java
Integer oldCount = playerViolationCount.put(
    playerId,
    playerViolationCount.getOrDefault(playerId, 0) + 1
);

if (oldCount != null
    && oldCount > kickViolationThreshold) {
    kick(player);
}
```

配置中的 `kick-violation-threshold` 是 `0`，但 `Map.put` 返回的是旧值。第一次触发事件时旧值为 `null`，不会进入踢出分支；第二次再触发时旧值为 `1`，才会超过阈值。也就是说，玩家实际上有且仅有一次违规事件的空间。

GrimAC 的两项关键配置均被改为：

```yaml
Simulation:
  immediate-setback-threshold: 0.5001
  max-advantage: 0.5001
```

前者控制单 tick 偏差达到多大时立即回退，后者控制累计优势。可利用的额外高度必须小于 `0.5001`，而越墙恰好需要约 `0.5` 格。

### 利用 1e-7 Stepping

Minecraft 的 MC-276267 碰撞问题俗称 `1e-7 Stepping`：在特定朝向和极小碰撞间距下，客户端的自动跨步逻辑会额外把玩家抬高 `0.5` 格。这个移动只需消耗第一次 `FlagEvent` 的容错，并且 `0.5 < 0.5001`，不会触发立即回退。

复现时要点如下：

1. 面向 $Z$ 轴负方向；
2. 靠近墙体调整到能触发行走辅助的极小距离；
3. 前进触发 stepping，使位置额外升高 `0.5` 格；
4. 立即正常跳跃；
5. 在跳跃最高点发送 `/flag`，并在接下来的 10 tick 内保持在墙顶以上。

另一种可行方式是在客户端制造幽灵方块，让自动跨步逻辑误判脚下碰撞，但本质仍是制造同一个 `0.5` 格 stepping 数据包。

### 自动发送命令

手动把 `/flag` 卡在最高点容易受到帧率和网络延迟影响。更稳定的做法是编写 Fabric 客户端 Mod，记录上一 tick 与当前 tick 的竖直速度；当速度由正变为非正时，说明刚越过跳跃顶点：

```java
if (lastYVelocity > 0 && currentVelocity.y <= 0) {
    client.player.networkHandler.sendChatCommand("flag");
}
lastYVelocity = currentVelocity.y;
```

满足两次高度检查后，服务端返回比赛 flag。附件里的 `config.yml` 将 flag 留空，因此不能从本地仓库直接读取真实值。

## 方法总结

决定性漏洞有两处：挑战插件误解 `Map.put` 的返回值，给第一次反作弊事件留下容错；GrimAC 阈值 `0.5001` 又刚好覆盖客户端 stepping 产生的 `0.5` 格偏差。完整解法需要反编译 Java 插件、理解反作弊配置并精确控制客户端运动，因此按主要障碍归入 Reverse。

出题人的 [r3craft 与 r4craft 分析](https://blog.wingszeng.top/2025-r3ctf-misc-r3craft-and-r4craft/) 还给出了插件源码、MC-276267 背景与 Fabric 自动化示例；正文已经写明不依赖外链即可复现的判断条件和时序。
