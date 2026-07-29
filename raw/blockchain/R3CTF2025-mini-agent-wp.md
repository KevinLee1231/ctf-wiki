# mini agent

## 题目简述

题目部署了一个持有约 500 ETH 的 `Arena`。选手 EOA 初始只有 8 ETH，通关条件是 `msg.sender.balance > 500 ether`。Arena 同时实现了存取款、ERC-20 风格余额和自动猪猪战斗，注册时还设置了两层限制：

- 玩家必须满足 `tx.origin == msg.sender` 且 `msg.sender.code.length == 0`；
- agent 运行时代码必须短于 100 字节，并禁止出现 `CREATE`、`CALL`、`CALLCODE`、`DELEGATECALL`、`CREATE2`、`SELFDESTRUCT` 的操作码字节。

完整利用链由 EIP-7702 代码委托、可预测随机数、存储槽预热和 `withdraw()` 中的重入下溢共同组成。

## 解题过程

### 用 EIP-7702 绕过 EOA 与 agent 限制

EIP-7702 允许 EOA 把执行逻辑委托给一个实现合约，同时交易中的 `tx.origin` 与 `msg.sender` 仍是该 EOA。授权账户上的委托标记形如：

```text
ef0100 || implementation_address
```

先为单独的 agent EOA 设置委托，再由尚未设置代码的玩家 EOA 调用 `register(agent)`。Arena 扫描到的只是 23 字节委托标记，而复杂逻辑位于实现合约中。由于实现地址的 20 个字节也会被逐字节检查，需要用 CREATE2 枚举 salt，直到地址不含 `f0`、`f1`、`f2`、`f4`、`f5`、`ff`。

玩家自身暂时保持无代码，以通过 `msg.sender.code.length == 0`；完成注册和战斗后，再给玩家 EOA 设置第二个 7702 委托，用于接收 ETH 时执行回调。

### 预测下一次战斗随机数

`Randomness.random()` 的更新式为：

```solidity
seed = uint256(
    keccak256(abi.encodePacked(block.prevrandao, msg.sender, seed))
);
```

agent 的 `tick()` 先主动调用一次 `random()` 获得当前新 seed。Arena 紧接着仍以自身地址为调用者再次调用同一函数，因此 agent 可以本地计算下一项：

```solidity
uint256 next = uint256(
    keccak256(abi.encodePacked(block.prevrandao, address(arena), seed))
) % 100;
```

把这个值作为 `tick()` 返回的随机参数，即可稳定获得伤害加成，赢下足够的赌注，使 Arena 内部余额达到至少一次提款所需的 10 ETH。

### 在 5000 gas 内完成重入

漏洞函数先转账、后扣款，而且扣款在 `unchecked` 中：

```solidity
payable(msg.sender).call{value: amount, gas: 5000}("");
unchecked {
    balanceOf[msg.sender] -= amount;
}
```

5000 gas 不足以访问冷存储并修改余额，但 EIP-2929 区分冷、热存储槽。先在正常调用中执行：

```solidity
arena.transfer(address(0), 1);
```

这样玩家和零地址对应的余额槽都已预热。随后令 `amount` 等于玩家当前 Arena 余额并调用 `withdraw(amount)`。玩家 EOA 收到 ETH 后执行 7702 委托的 `receive()`，在 5000 gas 内再次 `transfer(address(0), 1)`，使内部余额先减少 1。

外层 `withdraw()` 恢复后继续计算：

$$
(\text{balance}-1)-\text{balance}\pmod{2^{256}}=2^{256}-1
$$

玩家的内部余额由此下溢为极大值。最后以 `address(arena).balance` 为参数再次提款，抽空 Arena，EOA 余额便超过 500 ETH。

## 方法总结

本题要求把多个看似独立的限制串起来：7702 保留 EOA 调用语义但提供合约逻辑；公开可推进的随机状态让战斗可预测；EIP-2929 的热槽计费让 5000 gas 回调仍能写存储；最后由延迟更新和 `unchecked` 把一次减 1 放大成余额下溢。

外部参考中的关键 PoC、gas 计算和完整调用顺序已在正文展开；原文可见 [valgrind 的 mini agent 复现](https://www.valgrindc.tf/posts/mini-agent/)，EIP-7702 的委托格式可对照 [规范正文](https://eips.ethereum.org/EIPS/eip-7702)。
