# GlacierCTF 2024 drainme

## 题目简述

题目实现了一个按份额存取 ETH 的金库。`Setup` 先部署金库和 `SharesBuyer`，后者持有 100 ETH；攻击目标是同时清空两者余额。

漏洞由两处逻辑组合而成：合约允许通过 `SELFDESTRUCT` 强制转入 ETH，而存款时又用已经包含本次 `msg.value` 的实时余额计算份额并向下取整。攻击者先抬高金库余额，再让受害者用 100 ETH 买到 0 份额，最后凭唯一的 1 份额提走全部资产。

## 解题过程

### 1. 分析份额计算顺序

首次存款时，合约直接令 `shares = msg.value`。已有份额后的计算则是：

```solidity
shares = totalShares * msg.value / address(this).balance;
balances[msg.sender] += shares;
totalShares += shares;
```

Solidity 进入函数时，本次 `msg.value` 已经计入 `address(this).balance`。整数除法还会向下取整，所以只要分母足够大，就能让一笔真实存款对应 0 份额。

普通转账会被 `receive()` 拒绝，但这不能阻止 `SELFDESTRUCT` 强制注资：

```solidity
contract ForceSend {
    constructor(address payable target) payable {
        selfdestruct(target);
    }
}
```

### 2. 制造零份额存款

按官方 exploit 的顺序执行：

1. 向辅助合约提供 100 ETH，并通过 `selfdestruct` 强制送入金库；此时 `totalShares` 仍为 0。
2. 攻击者正常存入 1 wei，取得唯一的 1 份额。
3. 调用 `SharesBuyer.buyShares()`，使受害者把自己的 100 ETH 存入金库。

第 3 步计算时，金库余额已经超过 200 ETH，而总份额只有 1，因此：

$$
\left\lfloor\frac{1\times100\ \mathrm{ETH}}{200\ \mathrm{ETH}+1\ \mathrm{wei}}\right\rfloor=0.
$$

受害者余额归零，却没有稀释攻击者的份额。

### 3. 提走全部余额

取款额按当前总余额分配：

```solidity
value = shares * address(this).balance / totalShares;
```

攻击者仍持有全部 1/1 份额，调用 `withdrawEth()` 即可取走金库中的全部 ETH，同时满足 `Setup.isSolved()` 对两个余额均为 0 的要求。仓库中的 flag 为：

```text
gctf{pl34s3_g1v3_m3_m0r3_th4n_z3r0_sh4r3s}
```

## 方法总结

本题是典型的 vault inflation attack：资产余额可以脱离份额账本增长，而向下取整又允许“付钱但铸造 0 份额”。设计份额金库时应以存款前余额计算兑换率，拒绝 0 份额结果，并考虑强制转账造成的余额偏移；更稳妥的实现还会预置虚拟资产与虚拟份额，降低首存者操纵兑换率的能力。
