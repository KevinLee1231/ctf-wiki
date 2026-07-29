# r3reach

## 题目简述

r3reach 是一道以 Paper Minecraft 插件为载体的逻辑逆向题。题面要求在正常交互距离之外“触碰”名为 `Flag` 的村民，但附件 `R3Reach-1.0.jar` 中的真实判定并不检查一次远距离交互事件，而是每 tick 检查两个独立条件：

1. 玩家到村民的距离是否始终大于 3 格；
2. 村民的位置是否发生过变化。

插件还把玩家的 `ENTITY_INTERACTION_RANGE` 设为 0，直接发送攻击或交互数据包不是预期突破口。真正可利用的是 `/magic` 命令提供的速度变换，以及 Minecraft 在同一 tick 中处理移动包、碰撞、实体位移和插件定时任务的先后顺序。

之所以归入 reverse，是因为决定解法的工作是从 JAR 字节码恢复服务器判定和 `/magic` 的精确语义，再据此构造一组满足验证器的移动输入；它不是传统 native 内存破坏，也不是普通 Web 漏洞。

## 解题过程

### 1. 反编译插件并恢复胜利条件

附件 JAR 中只有一个主要类：

```text
com.r3kapig.r3reach.R3Reach
```

用 `javap -c -p` 或 Java 反编译器检查可得到以下关键状态：

```java
private Villager flagVillager;
private Location villagerLastLocation;
private boolean distanceValid;
private boolean gotFlag;
```

重置挑战时，插件把村民放在：

```text
(0.5, -59, 0.5)
```

玩家起点约为：

```text
(3.5, -59, 3.5)
```

同时执行：

```java
player.getAttribute(Attribute.ENTITY_INTERACTION_RANGE)
      .setBaseValue(0.0);
```

距离判断为：

```java
player.getLocation()
      .distanceSquared(flagVillager.getLocation()) > 9.0
```

也就是玩家必须离村民严格大于 3 格。插件通过 `runTaskTimer(..., 0L, 1L)` 每 tick 更新：

```java
distanceValid &= isDistanceValid(player);
```

这个布尔值是“粘住”的：只要任意一次检查发现玩家过近，之后即使再退远也不会恢复，必须执行 `/reset`。

胜利部分可整理为：

```java
if (distanceValid
        && !gotFlag
        && villagerLastLocation != null
        && flagVillager.getLocation()
              .distanceSquared(villagerLastLocation) > 1e-8) {
    gotFlag = true;
    showTitle("CONGRATULATIONS!", config.getString("flag"));
}
villagerLastLocation = flagVillager.getLocation().clone();
```

所以实际目标不是从远处触发交互，而是：

> 让物理碰撞移动村民，同时让插件每次取样时都只看到玩家位于 3 格之外。

### 2. 理解 `/magic` 的速度变换

`/magic` 保存当前速度，然后把竖直下落速度转为视线方向速度：

```java
Vector oldVelocity = player.getVelocity().clone();
double magic = -oldVelocity.getY();
player.setVelocity(
    player.getLocation().getDirection().multiply(magic)
);
```

两 tick 后，插件恢复旧速度：

```java
runTaskLater(plugin, () -> player.setVelocity(oldVelocity), 2L);
```

若玩家正在下落，则 `oldVelocity.getY()` 为负，`magic` 为正。只要先在平台外积累足够大的向下速度，再面向村民执行 `/magic`，就会获得一次短暂而强的水平推进。

### 3. 排除直接交互与普通靠近

几条直觉路线都会失败：

- 远距离发送 `Use Entity` 或攻击包：交互距离已被设为 0，而且村民为 invulnerable；
- 正常跑近撞击再离开：每 tick 的 `distanceValid &= ...` 会永久记录过近状态；
- 在平台上执行 `/magic`：没有足够下落速度，水平冲量太小；
- 从平台正下方竖直冲刺：上方方块阻挡，难以稳定制造村民位移。

因此必须同时控制“服务器认为玩家在哪里”和“客户端何时恢复正常发送移动包”。

### 4. 在平台外积累下落速度

公开复现使用的准备位置为：

```text
(4.35, -58.9, 0.5)
```

该点有三个性质：

- 到初始村民 `(0.5, -59, 0.5)` 的距离大于 3；
- X 坐标略微越过平台边缘，玩家会开始自由下落；
- 面向西侧时，`/magic` 生成的水平速度正对村民。

需要一个能够转发并改写 Minecraft 移动包的代理。先暂停普通客户端移动包，再向服务器注入位置与朝向，使玩家在上述位置停留约 4.8 秒。此时服务端持续计算重力，`oldVelocity.y` 的绝对值逐渐增大，而插件看到的距离仍合法。

### 5. 利用同 tick 包顺序完成碰撞后回退

准备好下落速度后：

1. 把客户端视图同步到伪造位置和面向方向；
2. 发送 `/magic`；
3. 恢复客户端移动包，使获得水平速度的玩家短暂冲向村民；
4. 在产生碰撞的近距离移动包后，立即追加一份远距离位置包；
5. 让插件定时任务取样时只看到已经回退的远距离坐标。

公开代理的核心控制序列如下，其中命令名属于该代理自身的控制接口：

```python
ctl("comp 1")
ctl("dropmove 1")
ctl("appendpos off")
ctl("reset")
time.sleep(1.0)

# 把服务器侧玩家放在平台外且仍满足距离要求的位置。
ctl("posrotid 31 4.35 -58.9 0.5 90 0 0")
time.sleep(4.8)

# 当客户端发出冲向村民的移动包时，紧接着追加远端位置。
ctl("appendposxlt -4.35 -58.9 0.5 1.3 2")

# 同步客户端、执行 /magic，然后放开移动。
ctl("cteleid 9900000")
ctl("ctele 4.35 -58.9 0.5 90 0", 0.010)
ctl("cmd magic", 0.050)
ctl("dropmove 0", 0.005)
```

这里真正依赖的不是这些辅助命令的名称，而是服务端收到的事件顺序：

```text
远距离自由下落
  -> /magic 把下落速度改为朝向村民的水平速度
  -> 近距离位置更新触发实体碰撞
  -> 村民被推动
  -> 同批次紧随一个远距离位置更新
  -> 插件下一次 tick 检查时玩家仍在 3 格外
```

网络抖动会影响 `/magic` 与放开移动的相对时间，实际利用应在小范围内重试多个延迟，而不是把 `0.050` 秒视为对所有环境都固定不变的常量。

### 6. 获取 flag

碰撞改变了村民位置，而 `distanceValid` 从未在插件采样时变为 false，下一次定时任务便向玩家显示动态 flag。公开远程复现得到：

```text
r3ctf{fuTURe_yoU-ReaCheD-Th3-F1ag_4Nd-rETUrnED_it-TO_pR3SeNt_you0}
```

包含代理控制命令和远程时序记录的公开题解见：[hax1ng 的 r3reach writeup](https://github.com/hax1ng/r3ctf-2026-writeups/blob/master/misc/r3reach.md)。本文已经根据本地 JAR 字节码补齐插件位置、距离阈值、每 tick 粘滞状态、两 tick 速度恢复与最终判定，外链用于保留代理实现语境和一次成功时序。

## 方法总结

r3reach 的漏洞本质是“连续安全属性”被离散采样实现。题目想保证玩家移动村民时始终位于 3 格外，但代码只在插件定时任务运行时取样位置；Minecraft 网络包处理和碰撞计算可以在两次取样之间短暂违反条件。

分析此类游戏协议题时，应把判定拆成三个时间轴：

- 客户端发送位置、朝向和命令的顺序；
- 服务端在一个 tick 内处理网络包、速度与碰撞的顺序；
- 插件定时任务读取状态的时刻。

只看最终坐标或只看交互距离都找不到解法。恢复 `/magic` 的速度变换后，关键是制造“中间状态推动目标、采样状态保持合法”的时间差，而不是突破服务器的 reach 校验本身。
