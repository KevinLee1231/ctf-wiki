# hideAndSeek

## 题目简述

Minecraft 1.19.2 服务器中，NPC Ben 每隔约 10 秒在：

```text
(0, -50, 0) 到 (128, 50, 128)
```

范围内重新出现，玩家可用 `/newtp x y z` 传送。随机传送和肉眼搜索成功率很低，因为 Ben 可能生成在方块内部并迅速消失。

关键是：服务器为了让客户端渲染实体，仍会把实体类型、坐标和 metadata 发给处于追踪范围内的客户端。无需从画面中看见 Ben，只要读取客户端已经收到的实体表即可。

## 解题过程

使用支持 Minecraft 1.19.2 的 Mineflayer 创建客户端 Bot，先传送到区域中心 `(64,0,64)`，尽量覆盖整个 $128\times128$ 范围：

```javascript
const mineflayer = require("mineflayer");

const bot = mineflayer.createBot({
  host: HOST,
  port: PORT,
  username: "Abot"
});

bot.once("login", () => {
  bot.chat("/newtp 64 0 64");
  setInterval(scanEntities, 350);
});
```

Mineflayer 会把服务端已同步的实体保存在 `bot.entities`。题目 NPC 是带自定义名称的实体，1.19.2 对应数据中名称可从 metadata 索引 2 观察。扫描时同时输出实体类型、ID、位置和名称：

```javascript
let handled = new Set();

function scanEntities() {
  for (const entity of Object.values(bot.entities)) {
    const customName = entity.metadata?.[2]?.toString();
    if (!customName || handled.has(entity.id)) continue;

    console.log({
      id: entity.id,
      type: entity.name,
      name: customName,
      pos: entity.position
    });

    if (customName.includes("Ben")) {
      handled.add(entity.id);
      const { x, y, z } = entity.position;
      bot.chat(`/newtp ${x} ${y} ${z}`);
    }
  }
}
```

实际版本若 metadata 索引不同，应先打印所有非空 metadata，而不是硬编码后认为“没有实体”。还要区分大厅中名为 Ben 的普通玩家与周期性新建/移动的 NPC：后者的实体 ID、类型、坐标和刷新节奏符合题面范围。

连接后站在中心轮询，可在 Ben 出现时立即读取其自定义名称或传送到其坐标完成交互。公开实例得到：

```text
R3CTF{Jus7_play_m0r3_h1de_2nd_seek_w1th_Ben}
```

Mineflayer 的完整连接与 metadata 轮询示例见 [R3CTF 2024 hideAndSeek 复现](https://blog.shenghuo2.top/posts/0bba261/)。本文已经补充实体去重、坐标输出、NPC/玩家区分和 metadata 版本差异处理。

## 方法总结

“不可见”或“生成在墙内”只限制正常渲染，不代表客户端没有收到实体状态。游戏客户端题应先检查本地实体表、网络包和调试日志，再考虑人工遍历地图。把玩家放在区域中心可以减少追踪距离问题，脚本轮询周期要明显短于 Ben 的存活周期，并对同一实体去重，避免 `/newtp` 刷屏触发服务器限速。
