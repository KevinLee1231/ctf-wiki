# r4craft

## 题目简述

`r4craft` 沿用 `r3craft` 的 Paper 1.21.6、GrimAC 与自定义挑战插件。服务端仍要求玩家从合法出生点出发，在 $Y\ge -56.7748$ 时发送 `/flag`，并在 10 tick 后保持 `blockY > -59` 且没有被踢出。

挑战插件同样错误地把 `Map.put` 的返回值当成新计数，因此第一次 `FlagEvent` 不会触发阈值为 `0` 的踢出逻辑。区别在于 GrimAC 配置进一步收紧：

```yaml
Simulation:
  immediate-setback-threshold: 0.25
  max-advantage: 0.25
```

`r3craft` 中产生 `0.5` 格偏差的 1e-7 Stepping 已经超过限制，必须逆向 GrimAC 的多项移动检查，只制造一次低于 `0.25` 的合法形态偏差。

## 解题过程

### 为什么不能额外发送一个移动包

直接伪造位置包通常会同时触发多种事件：

- `FlightA`：玩家并未处于飞行状态，却发送了不合规则的 flying 包；
- `GroundSpoof`：数据包声明的 `onGround` 与服务端预测不一致；
- `Simulation`：实收位置和物理预测位置存在偏差；
- `TickTimer`：同一 tick 出现多个 movement packet，或包序列与 `CLIENT_TICK_END` 不一致。

挑战插件只容许第一次事件。即使位置偏差只有 `0.24`，若额外插入一个包，后续由客户端重力计算产生的包还会继续触发事件，最终被踢出。因此不能“加发”数据包，而要修改客户端本来就会发送的某一个正常移动结果。

### 选择跳跃最高点

正常跳跃过程中，在空中发送的原生位置包本来就满足：

1. 玩家确实处于跳跃状态，不会被 `FlightA` 当作飞行；
2. `onGround=false` 与服务器状态一致，不触发 `GroundSpoof`；
3. 每 tick 仍只有正常数量的移动包，不破坏包序列。

跳跃最高点的竖直速度接近零，此时把客户端 $Y$ 坐标增加 `0.249`：

```java
Vec3d velocity = player.getVelocity();

if (velocity.y > 0 && velocity.y < 0.01 && enabled) {
    player.setPosition(
        player.getX(),
        player.getY() + 0.249,
        player.getZ()
    );
}
```

偏差满足：

$$
0.249 < 0.25
$$

它只触发一次 `Simulation` 事件，不达到立即回退阈值，也正好利用了挑战插件对首次事件的漏判。

### 在正确时刻请求 flag

Fabric Mod 继续记录竖直速度。当速度从正数变成非正数时，客户端刚通过最高点，自动发送命令：

```java
if (lastYVelocity > 0 && velocity.y <= 0) {
    player.networkHandler.sendChatCommand("flag");
}
lastYVelocity = velocity.y;
```

实际操作必须潜行跳跃。若普通跳跃，位置抬高后客户端还可能发送额外的后续移动包，造成新的预测偏差并触发第二次事件；潜行可以让这段包序列保持在预期形态内。

完整流程为：

1. 使用 Fabric 客户端进入服务器；
2. 开启位置修改功能；
3. 保持潜行并跳跃；
4. 在上升速度落入 $(0,0.01)$ 时把位置提高 `0.249`；
5. 在速度跨过零点时自动发送 `/flag`；
6. 保持在线等待 10 tick 后的第二次高度检查。

附件的 flag 配置为空，真实 flag 只能由比赛服务返回。

## 方法总结

本题不是单纯的“发一个更高坐标”，而是要求满足 GrimAC 的预测、地面状态和包序列三套约束。可行窗口是正常跳跃的最高点：修改已有包而不是追加包，把偏差控制在 `0.25` 以下，再利用挑战插件首次 `FlagEvent` 不踢人的逻辑错误。

出题人的 [r3craft 与 r4craft 分析](https://blog.wingszeng.top/2025-r3ctf-misc-r3craft-and-r4craft/) 提供了完整 Fabric 示例和涉及的 GrimAC 检查源码；正文已保留位置增量、速度条件、潜行要求和命令时序。
