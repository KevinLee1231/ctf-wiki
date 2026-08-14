# CakeCTF 2022 matsushima3 Writeup

## 题目简述

题目提供一个 Pyxel 二十一点客户端，实际牌局状态保存在 Flask/Redis 服务端。初始资金为 100，每次获胜翻倍；资金超过 `999999999999999` 时，响应中返回 flag。

服务存在两处可组合的逻辑缺陷：洗牌种子可由时间和公开的 `user_id` 预测；`/game/act` 把所有非 `hit` 动作当作站牌处理，却只在结算阶段把字面值 `stand` 认作站牌。自定义动作因此可以在不结束牌局的情况下消耗牌堆。

## 解题过程

### 恢复完整牌堆

创建用户时，服务返回 `user_id`。每局开始使用：

```python
random.seed(int(time.time()) ^ session['user_id'])
random.shuffle(deck)
```

`/game/new` 还会返回玩家起手两张牌。发牌顺序是玩家、庄家、玩家、庄家，所以这两张牌分别对应洗牌后列表的 `deck[-1]` 与 `deck[-3]`。

在当前时间附近枚举几秒，就能用这两个已知位置确认正确 seed：

```python
import random
import time

def recover_deck(user_id, player_hand):
    now = int(time.time())
    for seed_time in range(now, now - 5, -1):
        deck = [(i // 13, i % 13) for i in range(52)]
        random.seed(seed_time ^ user_id)
        random.shuffle(deck)

        if (deck[-1] == tuple(player_hand[0]) and
                deck[-3] == tuple(player_hand[1])):
            return deck, seed_time
    raise RuntimeError('没有在时间窗口内找到洗牌种子')
```

恢复牌堆后，庄家的暗牌和所有后续牌都变成已知信息。

### 利用未知 action 跳牌

服务端分支为：

```python
if player_action == 'hit':
    # 玩家拿牌
    ...
else:
    # 所有其他字符串都执行庄家站牌流程
    ...
```

但胜负比较却写成：

```python
player_action == 'stand'
```

因此提交 `action=xxx` 时：

- 若庄家分数不高于 16，庄家会先按规则拿牌；若爆牌，玩家直接获胜；
- 若庄家已经高于 16，服务仍会先 `pop` 一张牌，却不会把它加入任何手牌；
- 因为动作不等于 `stand`，庄家未爆牌时牌局仍保持 `game` 状态。

这就形成“跳过下一张牌”的能力。官方修改版客户端增加 `SKIP` 动作：

```python
class GameAction(enum.Enum):
    HIT = enum.auto()
    STAND = enum.auto()
    SKIP = enum.auto()

def notify_action(self, action):
    if action == GameAction.HIT:
        response = self.connection.request('game/act', {'action': 'hit'})
        self.deck.pop()
    elif action == GameAction.STAND:
        response = self.connection.request('game/act', {'action': 'stand'})
    else:
        response = self.connection.request('game/act', {'action': 'xxx'})
        self.deck.pop()  # 与服务端丢弃的 next_card 同步
```

客户端同时显示恢复出的庄家手牌和接下来两张牌。先用未知动作让庄家完成拿牌，再跳过不利牌；遇到能让玩家安全提高到庄家之上的牌时选择 `hit`，最后发送真正的 `stand` 完成结算。

### 累积奖金

每次胜利资金乘二。需要的胜局数为：

$$
100\times2^{43}=879609302220800<10^{15},
$$

$$
100\times2^{44}=1759218604441600>10^{15}.
$$

所以连续完成 44 次可控胜局后，服务响应中出现：

```text
CakeCTF{INFAMOUS_LOGIC_BUG}
```

## 方法总结

该题虽然带本地游戏客户端，但关键漏洞位于 HTTP 应用状态：可预测 PRNG 泄露整副牌，动作解析与结算判断不一致又提供跳牌原语，因此归入 Web。

服务端不应使用秒级时间与公开 ID 作为游戏随机种子，也不应以“不是 hit 就当 stand”的方式处理枚举值。应对 action 做严格白名单验证，并让状态转换和胜负结算使用同一个规范化动作值。
