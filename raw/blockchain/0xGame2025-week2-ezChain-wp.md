# ezChain

## 题目简述

题目给出一个 Solidity 挑战合约。合约初始状态为 `solved = false`，`solve` 会把传入字符串与常量 `welcome_to_0xGame2025` 分别经过 `abi.encodePacked` 和 `keccak256`，哈希相等时将 `solved` 置为 `true`；平台通过 `isSolved()` 检查结果。

关键逻辑等价于：

```solidity
function solve(string memory phrase) public {
    if (
        keccak256(abi.encodePacked(phrase)) ==
        keccak256(abi.encodePacked("welcome_to_0xGame2025"))
    ) {
        solved = true;
    } else {
        revert("Wrong phrase!");
    }
}
```

因此本题不涉及哈希碰撞或合约漏洞，目标只是向题目部署的合约实例发送一次正确调用。

## 解题过程

在 Remix 中以 Solidity 0.8.x 编译 `Setup.sol`，然后在 **Deploy & Run Transactions** 面板选择与题目网络一致的环境。若平台已经给出合约地址，应使用 **At Address** 连接该实例，不能另行部署一个与平台无关的新实例。

连接后调用：

```text
solve("welcome_to_0xGame2025")
```

等待交易上链，再调用 `isSolved()`；Remix 的返回值为：

```text
0: bool: true
```

这表明 `solved` 已被置为 `true`，挑战完成。

也可以使用 Foundry 的 `cast` 发送同一笔交易：

```bash
cast send <合约地址> \
  "solve(string)" "welcome_to_0xGame2025" \
  --rpc-url <RPC 地址> \
  --private-key <私钥>
```

其中合约地址、RPC 和私钥均应替换为题目实例提供的临时参数，不应写入题解。

## 方法总结

先直接阅读状态判定函数，确认平台真正检查的是 `isSolved()`。本题的输入明文已硬编码在合约中，只需连接正确的已部署实例并调用 `solve`；最容易出错的地方是误把“部署合约”当成目标，导致修改的是自己新建的实例而不是平台实例。
