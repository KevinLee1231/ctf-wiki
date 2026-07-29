# SEKAI Meme Launcher

## 题目简述

题目运行在以太坊主网分叉环境中。`MemeManager` 持有 100 ETH 的启动资金，用户可以创建 Meme 代币、参加预售、初始化 Uniswap V2 流动性，并通过管理器提供的自定义 `swap` 接口交易。目标是让玩家地址余额达到 50 ETH。

关键问题不在 AMM 数学本身，而在 `MemeManager.swap()` 的资金归属检查：买入分支会让路由器花费管理器自己的 ETH，却没有验证调用者附带的 `msg.value` 是否足以覆盖本次买入。

## 解题过程

### 1. 预售与初始化流动性

创建代币时可以指定初始价格。官方解法使用约 $10^{-4}$ ETH/Token 的价格，并先参加预售：

```solidity
manager.preSale{value: 0.95 ether}(meme, 9500 ether);
```

预售只检查价格等式以及最低付款额。随后调用初始化函数，管理器会从其风投资金中拿出 10 ETH，并配合 100000 枚代币建立交易对。此时攻击者已经持有 9500 枚可卖出的代币。

### 2. 找到 `swap` 的付款缺口

`swap()` 用紧凑字节流描述一组交易操作。买入分支最终调用路由器，并把 `amount` 作为 ETH 输入，但函数没有建立下面这个必要约束：

```solidity
require(msg.value >= totalEthInput);
```

函数虽然在解析过程中维护了类似“剩余付款”的状态，却没有在结束时要求它归零或非负。于是调用者可以用零或远小于买入额的 `msg.value` 构造操作，让管理器用自己的余额替攻击者买币。

利用时发送官方脚本所用的紧凑操作流，使输出代币直接进入玩家地址：

```solidity
manager.swap{value: 0}(packedOperations);
```

这样管理器的 ETH 被注入池子，池中的 ETH 储备上升，而攻击者额外得到 Meme 代币。

### 3. 把池中 ETH 卖回给玩家

接下来将攻击者持有的全部 Meme 代币授权给路由器，再走正常卖出路径换回 ETH：

```solidity
IERC20(meme).approve(address(router), type(uint256).max);
router.swapExactTokensForETHSupportingFeeOnTransferTokens(
    IERC20(meme).balanceOf(address(this)),
    0,
    path,
    address(this),
    block.timestamp
);
```

一次循环会把管理器的部分启动资金转化为玩家利润。因为每个新 Meme 都能建立独立池子，可以重复执行：

1. 创建低初始价格的新代币；
2. 用约 0.95 ETH 参加预售；
3. 让管理器出 10 ETH 初始化池子；
4. 用未付款的 `swap` 消耗管理器余额替自己买币；
5. 将持币全部卖回池子。

重复若干次后，把所得 ETH 转给玩家地址：

```solidity
payable(player).transfer(address(this).balance);
```

当 `player.balance >= 50 ether` 时，检查函数返回成功。

## 方法总结

这道题的核心是“执行交易的人”和“实际付款的人”发生了错位。自定义批处理或紧凑编码接口不能只在中间维护余额变量，还必须在入口和出口同时验证：

- 每笔外部调用的价值来源明确；
- 累计花费不超过调用者实际付款；
- 解析结束后不存在未结算的负债；
- 合约自有余额不会被当作用户交易的默认补贴。

利用链并不依赖预言机操纵或复杂闪电贷。预售只是低成本取得初始筹码，真正产生利润的是管理器替攻击者支付买入 ETH，随后攻击者再通过 AMM 把这部分价值兑现。
