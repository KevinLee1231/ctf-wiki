# h1de@ndSe3k

## 题目简述

第二阶段把 Ben 的活动范围扩大为：

```text
(0, -50, 0) 到 (512, 50, 512)
```

并给实体添加隐身效果；`/newtp` 也只能传送到这个范围内。肉眼随机搜索几乎不可行，但隐身只影响客户端渲染，不会让实体从协议和客户端世界对象中消失。核心仍是读取实体状态，只是需要主动移动追踪窗口覆盖更大的地图。

## 解题过程

### 用 Mineflayer 扫描追踪范围

沿 $x,z$ 平面建立间隔约 64 格的网格，在合法范围内节流传送。每次移动后检查 `bot.entities`：

```javascript
async function sweep() {
  for (;;) {
    for (let x = 0; x <= 512; x += 64) {
      for (let z = 0; z <= 512; z += 64) {
        bot.chat(`/newtp ${x} 0 ${z}`);
        await delay(700);

        for (const entity of Object.values(bot.entities)) {
          const name = entity.metadata?.[2]?.toString();
          if (name?.includes("Ben")) {
            const p = entity.position;
            console.log(name, p);
            bot.chat(`/newtp ${p.x} ${p.y} ${p.z}`);
            return;
          }
        }
      }
    }
  }
}
```

间隔应按服务器实体追踪距离调整；64 是公开复现中的有效经验值。延时过短会导致客户端尚未收到新区块实体，也可能因命令刷屏被 Paper 限速或踢出。Ben 会继续移动，所以扫描循环必须持续，并在命中后立即停止普通网格传送。

### 用改造客户端自动命中

另一种更稳定的方案是在 Meteor 一类 Fabric 客户端的 ESP/Tracer `onRender` 中遍历：

```java
for (Entity entity : mc.world.getEntities()) {
    Text custom = entity.getCustomName();
    if (custom == null || !custom.getString().contains("Ben")) continue;
    if (handledEntityId == entity.getId()) continue;

    handledEntityId = entity.getId();
    ChatUtils.sendPlayerMsg(
        "/newtp " + entity.getX() + " " +
        entity.getY() + " " + entity.getZ()
    );
}
```

必须按名称或实体特征过滤，并以实体 ID 做一次性保护。若对世界中的每个实体都发送命令，服务器很容易判定刷屏。客户端 `latest.log` 也会记录实体加入、metadata 更新和删除事件，即使 Ben 不参与正常渲染，日志仍可用来确认命中。

Ben 可能生成在方块内部。传送后可使用第三人称视角与 X-ray 查看名称标签，或直接根据客户端对象中的坐标和自定义名称完成交互。公开实例返回：

```text
R3CTF{You_2re_rea1ly_g00d_at_wr1ting_minecraft_sc2ip7s}
```

[Mineflayer 网格扫描复现](https://blog.shenghuo2.top/posts/0bba261/)和 [Meteor 客户端插桩记录](https://ecomaikgolf.com/posts/0008-r3ctf---h1dendse3k-writeup/)分别展示了两条可行路径。正文已经概括实体为何仍可见、如何覆盖 512 格范围、如何避免命令刷屏以及墙内生成的处理方式。

## 方法总结

隐身是显示层属性，不是网络层访问控制。只要服务器仍要求客户端追踪该实体，坐标、类型和 metadata 就会进入客户端内存。与第一阶段相比，本题新增的实际障碍是追踪范围而不是隐身本身，因此要用节流网格移动或客户端渲染钩子扩大观察窗口。自动化脚本的可靠性取决于过滤、去重、传送后等待和版本对应的 metadata 解析。
