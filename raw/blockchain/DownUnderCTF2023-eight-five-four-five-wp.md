# DownUnderCTF 2023 Eight Five Four Five Writeup

## 题目简述

这是一道 EVM 连接热身题。目标是让挑战合约的 `isSolved()` 返回真。构造函数保存了一个字符串，`solve_the_challenge` 会比较提交字符串与该值的 Keccak-256 哈希。

## 解题过程

变量 `use_this` 虽声明为 `private`，但合约主动提供了公开只读函数：

```solidity
function readTheStringHere() external view returns (string memory) {
    return use_this;
}
```

从实例 API 取得 RPC、玩家私钥和合约地址后，先读取该函数：

```bash
cast call <CONTRACT> "readTheStringHere()(string)" --rpc-url <RPC_URL>
```

返回内容是：

```text
I can connect to the blockchain!
```

用题目分配的玩家私钥发送状态修改交易：

```bash
cast send <CONTRACT> \
  "solve_the_challenge(string)" \
  "I can connect to the blockchain!" \
  --private-key <PLAYER_PRIVATE_KEY> \
  --legacy \
  --rpc-url <RPC_URL>
```

再次调用 `isSolved()(bool)` 会得到 `true`，随后从题目完成接口领取：

```text
DUCTF{I_can_connect_to_8545_pretty_epic:)}
```

## 方法总结

Solidity 的 `private` 只限制其他合约直接通过语言级接口访问，并不等于链上秘密；本题甚至提供了显式读取函数。解题重点是区分 `eth_call` 的只读查询与需要签名、消耗测试链 Gas 的状态交易，并确认交易使用题目发放的账户和 RPC。
