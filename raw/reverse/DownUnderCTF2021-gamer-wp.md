# DownUnderCTF 2021 - gamer

## 题目简述

题目是一个 Unity WebGL 平台游戏：先收集四枚金币，再前往地图最右侧取得 flag。尖刺、空中的金币、敌人群和极远的终点使正常通关很困难，但游戏逻辑完全运行在客户端；利用 Unity 暴露给 JavaScript 的消息接口和浏览器计时 API，即可修改生命、跳跃与游戏速度。

## 解题过程

### 确认逻辑位于客户端

浏览器网络面板只显示初始加载的 `game.data`、WebAssembly 和 JavaScript 文件，游玩过程中没有决定胜负的服务端请求。因此可以下载资源离线分析，也可以直接修改页面脚本。

页面通过 `createUnityInstance` 创建游戏实例。把返回对象保存到全局作用域，便能从控制台调用它：

```javascript
createUnityInstance(canvas, config, onProgress).then((unityInstance) => {
    window.unityInstance = unityInstance;
});
```

### 恢复对象名与可调用方法

在 `game.data` 的脚本元数据中可以找到：

```text
PlayerLogic HandleMovement CheckIfGrounded amt BoosterJump
UpdateHealth health moveSpeed jumpForce
```

结合 Unity 常见命名方式可判断对象名为 `Player`，有用的方法包括 `UpdateHealth` 与 `BoosterJump`。`SendMessage` 会把消息送到指定 GameObject 上的脚本方法，因此可直接执行：

```javascript
unityInstance.SendMessage("Player", "UpdateHealth", 100);
unityInstance.SendMessage("Player", "BoosterJump", 12);
```

第一条补充生命，使角色能够穿过尖刺或承受敌人伤害；第二条提供额外向上速度，用于越过障碍并取得高空金币。如果参数类型或数量不正确，Unity 控制台错误也会提示目标方法需要一个参数。

### 加速游戏时间

地图终点距离很远。Unity WebGL 使用 `performance.now()` 计算经过时间，可以保留真实函数，再返回按倍数放大的时间差：

```javascript
const originalNow = performance.now;
const realStart = originalNow.call(performance);

function speedhack(multiplier) {
    performance.now = () => {
        const realTime = originalNow.call(performance);
        return realStart + (realTime - realStart) * multiplier;
    };
}

speedhack(8);
```

这会整体加快依赖游戏时间的移动。配合补血和跳跃，依次收集四枚金币，再沿指示牌向右到达终点，即可取得：

```text
DUCTF{y0u_4r3_a_pr0_g4m3r_a38fb}
```

下图保留了修改后的完整游玩效果，能直观看到加速移动、强化跳跃以及绕过原有限制的过程：

![使用补血、强化跳跃和时间加速通关 Unity 游戏的动态演示](./DownUnderCTF2021-gamer-wp/hacked-game-playthrough.gif)

## 方法总结

本题的决定性问题是客户端游戏逻辑可被操纵。分析顺序是：确认没有服务端权威校验，暴露 Unity 实例，从资源元数据恢复对象和方法名，用 `SendMessage` 修改角色状态，再重载 `performance.now()` 缩短长距离移动时间。对于 WebGL 游戏，网络边界、引擎桥接接口和计时源通常比盲目修改 WebAssembly 内存更值得优先检查。
