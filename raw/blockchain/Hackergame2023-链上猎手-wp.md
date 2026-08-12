# 链上猎手

## 题目简述

题目在一条未启用 Shanghai 升级的 EVM 私链上部署两套原版 Uniswap V2 factory、WETH、一个 ERC-20 Token，以及两组 WETH/Token pair。两组池子的初始储备分别为 1:1 和 1:2，因此存在 `WETH -> Token -> WETH` 套利空间。链下 Bot 逐块枚举共同交易对、计算最优输入，并由 owner 账户调用链上 Bot 合约；链上合约初始持有 1 WETH。每一问都要求把该合约的 WETH 余额清空。

三问分别考查 callback 身份认证、链下 `eth_call` 模拟与下一块真实执行的环境差异，以及通用外部调用留下的 ERC-20 allowance。它们都依赖 EVM 调用上下文、交易原子性和链上持久状态，归为 `blockchain`。

## 解题过程

### The Maximal Extractable Value：伪造 pair callback

第一版 `uniswapV2Call` 只做两项检查：

```solidity
require(
    IUniswapV2Pair(msg.sender).factory() == FACTORY1 ||
    IUniswapV2Pair(msg.sender).factory() == FACTORY2
);
require(sender == address(this));
```

两项都不构成可信身份：恶意合约可以实现一个返回合法 factory 地址的 `factory()`；`sender` 只是调用者可控参数。回调随后从 `data` 解出 `pair1` 和 `amount1`，执行

```solidity
WETH.transfer(address(pair1), amount1);
pair1.swap(...);
```

因此部署一个同时实现 `factory()` 与空 `swap()` 的假 pair，令 `pair1=address(this)`、`amount1=Bot` 的全部余额，再直接调用 `uniswapV2Call`，WETH 就被转到恶意合约。正确校验应由可信 factory 对两种 token 计算或查询唯一 pair，并要求 `msg.sender` 精确等于该地址，不能相信调用者自报的接口结果。

### The Dark Forest：区分模拟与真实上链

第二版把 callback 检查改成：

```solidity
require(tx.origin == owner, "origin");
```

攻击者无法直接满足，但可以创建恶意 ERC-20，并分别在两套 factory 建池、按不同价格注入流动性。链下 Bot 会发现该套利机会，先在最新块状态上调用 `simulate(...).call()`，模拟成功后再签名发送 `arbitrage` 交易。于是恶意 Token 的 `transfer` 会同时在两种上下文中执行，而且真实交易由 Bot owner 发起，自动满足 `tx.origin`。

关键差异是：`eth_call` 使用当前最新块号，真实交易进入下一块。恶意 Token 可在确定的私链环境中用 `block.number` 区分二者：模拟时正常转账，让 `simulate` 看到 Bot 余额增加；真实上链时触发回调，把 Bot 的 WETH 转走。概念化逻辑如下：

```solidity
function transfer(address to, uint value) public returns (bool) {
    if (!hacked && block.number % 2 == REAL_TX_PARITY) {
        Bot(bot).uniswapV2Call(address(0), 0, 0, craftedData);
        hacked = true;
    }
    _transfer(msg.sender, to, value);
    return true;
}
```

具体掏空数额应由当前池储备和 Bot 余额计算，而不是长期保存官方实例地址或常量。安全上不能假设单独一次 `eth_call` 与未来上链完全等价；至少应在目标块上下文中模拟完整 bundle，并在合约内部强制执行资产不变量。

### Death's End：先授权，交易后再转走

第三版 `arbitrage` 在执行任意 `(target, calldata)` 列表前后检查 WETH 余额增加；callback 同样能执行任意调用：

```solidity
for (uint i = 0; i < addressList.length; i++) {
    (bool success, ) = addressList[i].call(calldataList[i]);
    require(success);
}
```

若在同一套利交易内直接转走 WETH，余额检查会回滚。但 ERC-20 `approve(spender, amount)` 只修改 allowance，不立即改变余额。构造恶意 Token 和交易对吸引 Bot 后，在 callback 中让 Bot 调用 WETH：

```solidity
WETH.approve(attacker, type(uint256).max);
```

套利交易结束时 Bot 余额仍未减少，因而通过后置检查；授权却已经持久化。攻击者随后另发一笔交易执行

```solidity
WETH.transferFrom(bot, attacker, WETH.balanceOf(bot));
```

即可清空余额。这说明只比较单个资产的即时 balance 不能约束一段通用调用的长期副作用，allowance、operator、授权签名和其他合约状态也必须纳入能力边界。

官方 `bot.py`、三版 Bot 合约、三份攻击合约和交易构造脚本相互印证了上述资产流。本轮只做静态复核，没有启动 Geth、部署合约或发送交易；复现时应固定 `evm_version=paris` 或使用兼容编译器，避免生成私链不支持的 `PUSH0`。

## 方法总结

- 核心技巧：分别利用可伪造的 callback 身份、模拟与真实交易的环境分歧，以及授权状态与余额检查之间的时间差。
- 识别信号：Uniswap 风格 callback、`tx.origin` 授权、链下 `eth_call` 决策、任意 `address.call(data)` 和仅检查余额前后差值，都是高风险组合。
- 复用要点：callback 必须从可信 factory 验证真实 pair；链上合约自己执行最终不变量；通用调用器应使用最小目标/selector 白名单，并审计 allowance 等延迟生效的持久状态。
