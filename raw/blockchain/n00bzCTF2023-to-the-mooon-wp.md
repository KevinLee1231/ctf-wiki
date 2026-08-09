# To_the_MOOON!

## 题目简述

合约用代币总供应量决定价格，但部署在 Solidity 0.6：整数运算默认不检查溢出。`buy` 只校验用户声称购买的 `_amount`，实际扣减量却由 `msg.value / price` 决定，因此可以让供应量下溢，再把价格更新为极大值。

## 解题过程

初始 `supply` 与 `price` 都是 10，关键代码为：

```solidity
function buy(uint _amount) public payable {
    require(_amount <= supply);
    uint _tokensBought = msg.value / price;
    balanceOf[msg.sender] += _tokensBought;
    supply -= _tokensBought;
}

function setPrice() public {
    price = supply * 1 wei;
}
```

在 Remix 中向 `buy(0)` 附带 110 Wei。声明购买量 0 通过检查，但实际 `_tokensBought` 为 11，于是 `10 - 11` 在 `uint256` 中回绕成 $2^{256}-1$。随后调用 `setPrice()`，`price` 变为这个极大值，满足 `solve()` 的 `price > 11 wei`。

调用 `solve()` 后记录交易哈希。题目给出的预期哈希为：

```text
0x9a51d4efe2f23f207e48c5b0f7463d7349150059603809493ce1669e73cdbd94
```

按题面要求去掉 `0x`，把哈希字符串的 ASCII 字节与给定十六进制掩码异或。输出会开始重复第二份 flag，截取第一段完整的 `n00bz{...}` 可还原：

```text
n00bz{why_wont_my_wallet_underflow}
```

## 方法总结

漏洞来自“校验的数量”和“实际结算的数量”不一致，并被 Solidity 0.6 的无检查算术放大为下溢。新版本可用内置溢出检查，但仍应让业务校验直接约束最终参与状态更新的值。
