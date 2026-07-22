# Viper

## 题目简述

题目给出一个由 Vyper `^0.2.16` 编译的双资产池，资产为 ETH 与 VETH，兑换比例 `ratio = 2`。合约用内部映射记录每个用户的两类余额，并提供 `deposit`、`withdraw`、`swap`；当合约自身 ETH 余额变为 0 时，`isSolved()` 返回真。

三个状态修改函数都写有 `@nonreentrant('lock')`，表面上已经做了跨函数重入保护。然而 Vyper 0.2.15、0.2.16 与 0.3.0 的存储槽分配缺陷会让不同函数中的同名锁落到不同 storage slot，实际只阻止重入同一个函数，不能阻止 `swap` 回调进入 `deposit`。

## 解题过程

### 定位外部调用与记账顺序

VETH 换 ETH 的分支先记录合约的 VETH 余额，然后在收到 `msg.value` 时把这笔 ETH 原样 `raw_call` 回调用者，之后才拉取 VETH 并增加用户的 ETH 内部余额：

```vyper
_before: uint256 = ERC20(self.coins[in_index]).balanceOf(self)
if msg.value > 0:
    raw_call(msg.sender, b"", value=msg.value)
ERC20(self.coins[in_index]).transferFrom(msg.sender, self, amount)
_after: uint256 = ERC20(self.coins[in_index]).balanceOf(self)
increase: decimal = convert(_after - _before, decimal) / self.ratio
self.balances[out_index][msg.sender] += convert(increase, uint256)
```

若锁正常工作，`raw_call` 期间调用者不能进入其他受同一锁保护的函数。[Vyper 当前文档](https://docs.vyperlang.org/en/stable/control-structures.html#nonreentrancy-locks)说明，`@nonreentrant` 的预期语义是所有受保护函数共享全局锁：进入函数时把专用 storage slot 设为 locked，回调进入任何其他受保护函数都应回滚。题目所固定的易受影响编译器版本破坏了这一前提，所以源码中“每个函数都写了同一个锁名”反而是关键识别信号。

### 在退款回调中存回 ETH

攻击合约先准备并授权一定数量的 VETH，然后带 $x$ wei 调用 `swap(1, 0, amount)`。池子进入 `else` 分支后把 $x$ wei 退回攻击合约，触发 `receive`/fallback。回调中立即执行：

```solidity
receive() external payable {
    pool.deposit{value: msg.value}(0, msg.value);
}
```

`deposit(0, x)` 的 `msg.value` 正好等于 `amount`，不会再触发退款分支，只会把 $x$ 计入攻击者的 ETH 内部余额。返回原来的 `swap` 后，池子又根据实际转入的 VETH 增加

$$
\frac{\text{amount}}{\text{ratio}}=\frac{\text{amount}}{2}
$$

的 ETH 内部余额。因此一次调用结束后，攻击者实际向池子净存入 $x$ wei，却获得

$$
x+\frac{\text{amount}}{2}
$$

的可提取 ETH 额度，凭空多出 `amount / 2`。核心原因是回调中的 `deposit` 记账与外层 `swap` 记账各自认为自己独占执行，实际却被错误锁槽交错执行。

攻击入口可以概括为：

```solidity
function attack(uint256 amount) external payable {
    veth.approve(address(pool), amount);
    pool.swap{value: msg.value}(1, 0, amount);
    pool.withdraw(0, pool.balances(0, address(this)));
}
```

按池中可用 ETH 和已持有 VETH 调整金额、必要时重复上述过程，最终把池子的实际 ETH 余额提空，使 `isSolved()` 成立。原始资料给出的验证 flag 为：

```text
ACTF{8EW@rE_0F_vEnom0us_sNaK3_81T3$_as_1t_HA$_nO_cOnSc1ENCe}
```

## 方法总结

- 核心技巧：利用 Vyper 易受影响版本的非重入锁槽分配错误，在 `swap` 的 ETH 退款回调中跨函数重入 `deposit`，制造重复内部记账。
- 识别信号：合约依赖编译器生成的重入锁，且关键函数在状态更新前执行 `raw_call`；此时必须核对具体编译器版本和实际 storage layout，不能只看源码装饰器。
- 复用要点：重入收益取决于“真实资产流”和“内部余额流”的差值。应逐步列出每次转账与每次记账，确认回调结束后凭空增加的余额，而不是笼统地把所有外部调用都称为可重入。
