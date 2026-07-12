# claim-guard

## 题目简述

题目由一个简单智能合约和一个 front-running bot 组成。bot 监听 pending transaction，模拟交易；如果交易包含有效 PoW，就把 gas price 翻倍并抢先提交。

官方提到部分代码来自真实 MEV bot，例如 burberry 类项目。相关外链/项目的重要信息是：真实 MEV bot 常做 pending tx 模拟、重放和抢跑，但模拟环境与真实出块环境存在差异，RPC 请求量、nonce 顺序和交易排序策略也会引入可利用的边界。

官方 WP 只给出 `burberry` 这一类真实 MEV bot 名称，没有给出具体 URL；因此正文保留模型和行为边界，不补无来源链接。

## 解题过程

### 关键机制

可利用点有三类。

#### 模拟区块与真实区块不一致

bot 模拟 pending tx 时通常使用上一块的环境信息。真实 proposed block 的 base fee 可能因为区块未满而下降。若构造 `solvePow` 交易，使其 gas price：

$$
\text{nextBaseFee} < \text{gasPrice} < \text{currentBaseFee}
$$

则 bot 用当前 base fee 模拟会失败，而真实下一块中该交易可被打包成功。

#### 与 bot 竞速

bot 需要对 pending tx 做模拟，如果交易执行过程中触发大量 `SLOAD`，会制造大量 RPC 请求并拖慢 bot。选手可先提交一个拖慢 bot 的交易，再提交同 nonce、更高 gas price 的 `solvePow` 交易，利用 nonce 替换和出块时序抢在 bot 前面。

#### 抬价压过 `registerBlock`

这是非预期。出题时假设 anvil 按 `gas * gas_price` 排序，实际只按 `gas_price`。玩家余额足够时，可以直接把 `registerBlock` 交易的 gas price 顶掉。bot 后续交易虽然 gas price 更高，但 nonce 小于 `registerBlock`，会因 nonce 顺序排在后面。

### 求解步骤

预期一：利用 base fee 差异。

```text
1. 计算当前 base fee 和下一块可能 base fee。
2. 构造有效 solvePow。
3. 设置 gasPrice 低于当前 base fee、高于下一块 base fee。
4. bot 模拟失败，不会抢跑；节点在下一块接受交易。
```

预期二：拖慢 bot 并替换 nonce。

```text
1. 部署/调用一个执行大量 SLOAD 的合约，增加 bot 模拟耗时。
2. 发送该慢交易。
3. 使用同 nonce 发送 solvePow，gas price 更高。
4. 出块时同 nonce 高价交易替换低价交易，solvePow 成功。
```

非预期：压过 `registerBlock`。

```text
1. 观察 bot 的 registerBlock 交易。
2. 发送 gasPrice 更高的交易。
3. anvil 按 gasPrice 排序，玩家交易排到 registerBlock 前。
4. bot 后续交易受 nonce 顺序影响，无法提前执行。
```

## 方法总结

- 本题不是合约漏洞，而是 MEV bot 的模拟与交易排序假设漏洞。
- 预期解一利用 base fee 预测差异。
- 预期解二利用大量 `SLOAD` 放大 bot 模拟延迟。
- 非预期利用 anvil 交易排序只看 `gas_price` 的实现细节。
