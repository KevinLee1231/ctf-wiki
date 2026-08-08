# GuessOneGuess

## 题目简述

Socket.IO 猜数游戏为每个连接维护 `totalScore`，只有在猜中时且 `totalScore > Number.MAX_VALUE` 才发送 flag。惩罚响应事件却直接执行 `totalScore -= data.score`，没有验证类型或范围。关键是将 score 累加为 JavaScript `Infinity`，而不是反复正常猜数。

## 解题过程

### 制造 Infinity

```js
socket.on('punishment-response', data => {
  totalScore -= data.score;
  guessCount = 0;
  targetNumber = Math.floor(Math.random() * 100) + 1;
});
```

不能发送数值 `-Infinity`：Socket.IO 的 JSON 表示会把 `Infinity`、`-Infinity`、`NaN` 变为 `null`。但服务端 `-=` 会做类型转换，因此可发送字符串 `"-Infinity"`，或连续两次发送有限 `-1e308`，使 `totalScore` 溢出为正 `Infinity`。例如：

```js
socket.emit('punishment-response', {score: '-Infinity'});
```

随后在 1--100 中猜中当前数字即可。每次失败响应提供“太小/太大”，所以可二分；题目范围很小，顺序枚举也足够。猜中后 `Infinity > 1.7976931348623157e308` 为真，响应的 `message` 含 flag。

### 验证

应观察惩罚后的 `game-message.score` 为可序列化表现，并在下一次 win 的 `showFlag` 为 true。该结论由 `game-ws.js` 静态核对；没有向比赛服务建立 Socket.IO 连接。

## 方法总结

- 核心技巧：利用前端可控、未校验类型的减法把分数转为 Infinity，再满足不可能的阈值比较。
- 识别信号：服务端将网络字段直接参与算术、阈值等于 `Number.MAX_VALUE`，以及 JSON 无法直接表达非有限数。
- 复用要点：按协议边界分别分析 JSON 序列化和 JavaScript 运行时强制转换；服务端应检查 `Number.isFinite`、类型及上下界。
