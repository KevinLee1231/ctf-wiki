# Simple AMM Vault

## 题目简述

SimpleVault 初始有 1000 份 SV、2000 GREY，share price 为 2；AMM 则持有 1000 SV 与 2000 GREY。AMM 的不变量不是普通乘积，而是按当前 vault 价格计算：

$$
k=x+\left\lfloor\frac{y}{p}\right\rfloor,
$$

其中 $x$ 是 SV 储备、$y$ 是 GREY 储备、$p$ 是每份 SV 对应的 GREY。目标是利用闪电贷清空 vault，使 `totalAssets == totalSupply == 0`，再以更低价格重新初始化 shares，从而改变 AMM 对同一储备的估值。

## 解题过程

玩家先领取 1000 GREY，然后从 AMM 闪电贷出全部 1000 SV。AMM 只转出 token，并未在 callback 前修改记录的 `reserveX`：

```solidity
amm.flashLoan(true, 1000e18, "");
```

回调中把 1000 SV 全部赎回。由于旧价格为 2，可取出 2000 GREY，并同时令 vault 的 `totalAssets` 与 `totalSupply` 都归零。接着只存回 1000 GREY：

```solidity
vault.withdraw(1000e18);       // 得到 2000 GREY
grey.approve(address(vault), 1000e18);
vault.deposit(1000e18);        // 重新得到 1000 SV
vault.approve(address(amm), 1000e18);
```

空 vault 的 `toSharesDown()` 按 1:1 铸币，因此 share price 被从 2 重置为 1。归还 1000 SV 后，AMM 的记账储备仍为 `(1000 SV, 2000 GREY)`，但按新价格计算的价值由 2000 上升到 3000，闪电贷后的 `k >= oldK` 检查反而通过。

现在用 0 个 SV 输入换出 1000 GREY：

```solidity
amm.swap(true, 0, 1000e18);
```

新储备为 `(1000, 1000)`，在 $p=1$ 时计算出的 $k$ 恰好回到 2000，仍不小于初始值。攻击合约还保留赎回后未重新存入的 1000 GREY，最终可转走足够资产并取得：

```text
grey{vault_reset_attack_a3e7a42b511cf0a8}
```

## 方法总结

当 AMM 把外部 vault 的可变 share price 纳入不变量时，价格本身就是可操纵状态。只检查交易前后按“当前价格”计算的 $k$，无法保证真实资产守恒。vault 在总资产和总份额同时归零后按 1:1 重启，更让攻击者能通过一次全额闪电贷重置价格。设计上应固定估值基准、限制空仓重置，或在同一原子操作中按真实资产余额校验偿付能力。
