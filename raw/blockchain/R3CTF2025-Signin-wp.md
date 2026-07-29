# Signin

## 题目简述

`Setup` 总共铸造 1000 LING：选手可领取 1 LING，剩余 999 LING 由 `solve()` 存入一个简化的 ERC-4626 Vault。通关条件是 Setup 用 999 LING 换到的 vLING 少于 500：

```solidity
vault.deposit(999 ether, address(this));
require(vault.balanceOf(address(this)) < 500 ether);
```

Vault 的份额计算为：

$$
\text{shares}=\left\lfloor
\frac{\text{assets}\cdot\text{totalShares}}{\text{totalAmount}}
\right\rfloor
$$

漏洞位于借款归还逻辑：归还时 1% 手续费只增加 `totalAsset.amount`，不会铸造新份额。选手可以用同一笔资金循环借还，把每份 vLING 的价格抬高。

## 解题过程

### 建立少量初始份额

先领取 1 LING，向 Vault 存入略小于 0.5 LING，例如 $d=0.49$ LING。空 Vault 按 1:1 铸造，因此：

$$
\text{totalAmount}=d,\qquad \text{totalShares}=d
$$

钱包中还保留 $1-d$ LING，用于支付后续手续费。

### 用借还循环抬高份额价格

每轮借出 Vault 当前可借的全部资产 $A$，再调用：

```solidity
repayAssets(A);
```

实际转回的是 $1.01A$，但状态更新是：

```solidity
totalAsset.amount += A / 100;
```

因此一轮结束后，借款状态清零，Vault 的记账资产变成 $1.01A$，份额总量仍为最初的 $d$。重复这一过程时，资产量近似按 $1.01$ 倍增长：

$$
A_n=d\cdot1.01^n
$$

增长部分全部由选手钱包中的剩余 LING 支付。由于系统总共只有 1 LING 可被选手控制，循环应在 `totalAsset.amount` 接近但不超过 1 LING 时停止。选择 $d<0.5$ 后，可以达到：

$$
\frac{\text{totalAmount}}{\text{totalShares}}
\approx\frac{1}{d}>2
$$

实现时每轮都读取 `totalAssets()`，借出该数值并批准 `amount + amount / 100` 后归还；同时留意整数除法产生的 wei 级舍入。

### 触发 Setup 的大额存款

份额单价超过 2 LING 后调用 `Setup.solve()`。Setup 存入 999 LING 时得到：

$$
\text{minted}
=\left\lfloor
\frac{999\cdot d}{A_n}
\right\rfloor
<500
$$

于是检查通过，`solved` 被置为 `true`。

## 方法总结

本题不是传统的“首存捐赠”攻击，而是借款手续费的会计处理破坏了资产与份额的同步关系。手续费属于 Vault，却没有对应铸造份额，因而全部价值都归已有份额持有人。

审计份额型金库时，应同时检查存款、捐赠、借款、还款和手续费是否一致更新 `totalAssets` 与 `totalSupply`。本题源码已足以推导完整循环；辅助说明可参考 [ERC-4626 份额价格操纵复现](https://blog.pradeepbhattarai.me/ctf-challenge-writeup-manipulating-an-erc4626-vault/)。
