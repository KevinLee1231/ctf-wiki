# Empty Vessel

## 题目简述

题目核心是一个自定义 `INR` 代币和 `Stake` 金库的组合题。`Setup.solve` 的通过条件不是直接要求拿到某个标志文本，而是要求：

1. 已执行 `stakeINR`（`staked == true`）。
2. 计算 `setup.stake.redeemAll(address(this), address(this))` 返回的资产 `assetsReceived` 不超过 `75_000 ether`，否则回退 `Setup__Chall__Unsolved`。

源码里暴露了两个关键机制：

- `INR.batchTransfer` 使用汇编手写转账，`mul(receivers.length, amount)` 的结果用于余额比较与后续扣款判断。
- `Stake` 的充值/赎回完全按份额公式走：`convertToShares`、`convertToAssets`、`redeemAll`（按比例销毁全部份额）。

官方脚本里已经给了完整 exploit：`admin/exploit/Solve.s.sol`。

## 解题过程

### 关键观察

在 `INR.batchTransfer` 中：

- 合约先把 `from` 余额读到 `ptr`。
- 用 `mul(receivers.length, amount)` 计算总支出并检查发送者余额。
- 循环中先给每个接收者分别增加 `amount`。
- 循环结束后才从发送者余额扣除同一个乘积 `receivers.length * amount`。

因为是在 `assembly` 中直接做 `sub`，`amount` 被构造得很大会触发取余（模 $2^{256}$）而不是自然报错。

令接收者数为 `2`，官方脚本取：

$$
\text{amount} = \left\lfloor\frac{2^{256}-1}{2}\right\rfloor+1 = 2^{255}
$$

则：

$$
2 \times \text{amount} \equiv 0 \pmod{2^{256}}
$$

因此余额检查看到的总支出是 $0$，最终扣款也是 $0$；但循环仍会逐个给接收者增加 $2^{255}$。把攻击合约自身列为一个接收者后，它会凭空获得巨额 INR。漏洞本质是“溢出的聚合检查/扣款”与“未聚合的逐项入账”不一致，并不是循环中逐次对发送者做大额减法。

### 实际利用链路

参考官方 `Solve.s.sol`，大致流程如下：

1. `setup.claim()` 获取基础 `bonusAmount`，准备初始可交互余额。
2. 设 `stakeAmount = 1`，`inflationAmount = 50_000 ether`，`amount = 2^255`。
3. `inr.batchTransfer([address(this), address(0)], amount)` 使聚合总额在模 $2^{256}$ 下变为 $0$，同时仍给两个接收者逐项记入巨额余额。
4. `inr.approve(stake, stakeAmount)`，`stake.deposit(stakeAmount, address(this))`，让攻击合约持有一个很小的 `vINR` 份额前提。
5. `inr.transfer(stake, inflationAmount)` 向金库注入额外波动（配合初始金库状态形成错误份额比例窗口）。
6. 调用 `setup.stakeINR()` 让题目环境进入已质押状态。
7. 调用 `setup.solve()`，此时 `redeemAll` 逻辑在被污染的份额/库存关系下给题目地址返回满足阈值的金额，解出题目。

`solve` 中的关键分支是：

```solidity
if (assetsReceived > 75_000 ether) revert Setup__Chall__Unsolved();
```

也就是说通过故意破坏代币余额一致性，使 `assetsReceived` 落在可通过的范围内即可通过。

### 关键代码（节选）

```yul
// Handout/src/INR.sol::batchTransfer（等价伪代码）
let total := mul(mload(receivers), amount)
if lt(senderBalance, total) { revert(0, 0) }
for { let i := 0 } lt(i, mload(receivers)) { i := add(i, 1) } {
    // 每个接收者仍单独增加 amount
    sstore(receiverSlot, add(sload(receiverSlot), amount))
}
sstore(senderSlot, sub(sload(senderSlot), total))
```

```solidity
// admin/exploit/Solve.s.sol
uint256 amount = ((type(uint256).max) / 2) + 1;
inr.batchTransfer(Receivers, amount);
```

## 方法总结

- 核心技巧：`assembly` 自定义转账若有算术边界不匹配（尤其在 `amount * len`、`sub`/余额更新上），可直接构造溢出窗口。这里是“检查值与实际扣款模型不一致”型失衡。
- 识别信号：当出现手写算术、`mul`/`sub` 在模 $2^{256}$ 下运行且没有防护时，要优先考虑 `balance*len`、循环扣款的溢出分支。
- 复用要点：同一类题要先审计 `check_expr` 与状态变更表达式是否一致；只要检测式与扣款式不等价，就可能用极值金额构造过小的检查通过却带来超额扣减效果。
