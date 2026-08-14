# bi0sCTF 2022 Snek Game Writeup

## 题目简述

Snek Game 是一个 $31\times31$ 的 WebSocket 贪吃蛇游戏。服务端初始蛇头位于 $(26,26)$，身体位于 $(26,27)$，首个樱桃位于 $(7,7)$；分数达到

$$
31\times31-2=959
$$

时返回 flag。

虽然交互通过 WebSocket 完成，但题目没有利用 HTTP、浏览器信任边界或服务端漏洞。核心是构造覆盖路径并自动发送移动序列，因此不应仅因出现 WebSocket 就归为 web；现有归档类别中也没有适合的通用算法目录，故放入 `_unclassified`。

## 解题过程

### 阅读服务端状态机

服务端允许的字符只有 `u`、`d`、`l`、`r`，但一次 WebSocket 消息可以包含任意长度的方向串：

```javascript
ws.on("message", message => {
    movement(ws.game, message.toString());
    ws.send(JSON.stringify(ws.game));
});
```

`movement` 会逐字符执行整个字符串，最后才回送一次状态。这使客户端可以把一整段确定性路径作为单条消息提交，而不必每 250 ms 模拟一次按键。

樱桃也不会出现在完整棋盘上。`offset=3`，候选坐标满足 $3\le x,y<28$，即只在中间 $25\times25$ 区域生成。胜利目标却接近填满整个 $31\times31$ 棋盘，因此路径必须同时覆盖樱桃候选区、使用外侧三格边框，并让蛇尾在最后阶段仍为下一次生成留下位置。

### 构造交替收缩的覆盖路径

官方解法先从 $(26,26)$ 向上 26 格、向左 26 格，把蛇头移到左上角附近；随后交替执行两条近似 Hamilton 路径。两条路径的横纵行程逐层缩短，转弯位置略有错开，避免不断增长的身体封死尚未访问的格子。

下面是原求解器中两条路径的完整生成逻辑，方向先用大写构造，发送前统一转成小写：

```javascript
function buildPaths() {
    let path1 = "D";
    let n1 = 29, n2 = 30;

    for (let i = 0; i < 9; i++) {
        path1 += "D".repeat(n1) + "R".repeat(n2) + "U";
        n1--; n2--;
        path1 += "L".repeat(n2) + "U".repeat(n1) + "R";
        n1--; n2--;
    }
    n1--; n2--;
    for (let i = 0; i < 5; i++) {
        path1 += "D".repeat(n1) + "R" + "D" + "R".repeat(n2) + "U";
        n1--; n2--;
        path1 += "L".repeat(n2) + "U" + "L" + "U".repeat(n1) + "R";
        n1--; n2--;
    }
    path1 += "RDRUU" + "L".repeat(30);

    let path2 = "D";
    n1 = 29; n2 = 30;
    for (let i = 0; i < 8; i++) {
        path2 += "D".repeat(n1) + "R".repeat(n2) + "U";
        n1--; n2--;
        path2 += "L".repeat(n2) + "U".repeat(n1) + "R";
        n1--; n2--;
    }
    path2 += "D".repeat(n1) + "R".repeat(n2) + "U";
    n1--; n2--;
    path2 += "L".repeat(n2 - 1) + "U" + "L" + "U".repeat(n1 - 1) + "R";
    n1--; n2--;
    for (let i = 0; i < 5; i++) {
        path2 += "D".repeat(n1 - 1) + "R" + "D" + "R".repeat(n2 - 1) + "U";
        n1--; n2--;
        path2 += "L".repeat(n2 - 1) + "U" + "L" + "U".repeat(n1 - 1) + "R";
        n1--; n2--;
    }
    path2 += "RDRUU" + "L".repeat(30);

    return [path1.toLowerCase(), path2.toLowerCase()];
}
```

### 按响应节奏自动发送

把脚本放在题目页面控制台或本地副本中。每收到一次服务端状态后再发送下一段，可以避免依赖浏览器发送缓冲区的时序：

```javascript
const socket = new WebSocket(location.href.replace(/^http/, "ws"));
const [path1, path2] = buildPaths();
const queue = ["u".repeat(26) + "l".repeat(26)];

for (let i = 0; i < 150; i++) {
    queue.push(path1, path2);
}

socket.onmessage = event => {
    const game = JSON.parse(event.data);
    console.log("score:", game.score);

    if (game.flag) {
        console.log(game.flag);
        return;
    }
    if (game.error) {
        throw new Error(game.error);
    }
    if (queue.length > 0) {
        socket.send(queue.shift());
    }
};
```

交替路径反复扫过樱桃可能出现的区域，并随着身体增长调整剩余空间。达到 959 分后连接返回并关闭：

```text
bi0sctf{F4NGT4ST1C_J0B}
```

## 方法总结

解题关键来自三处服务端事实：消息可携带整串方向、樱桃只在三格边框以内生成、胜利要求 959 分。先围绕这些约束设计路径，再把人工按键改成按响应推进的 WebSocket 队列，就能稳定完成游戏。

本题也说明分类应依据决定性障碍而不是传输协议。WebSocket 只是自动化通道；没有漏洞利用、鉴权绕过或浏览器攻击面时，把它归为 web 会误导后续检索。
