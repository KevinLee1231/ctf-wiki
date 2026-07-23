# Outer Stellar

## 题目简述

题目实现了一座连接 Sui Move 与 Stellar Soroban 的跨链桥。玩家可在两条链各领取 100 枚 SEKAI，目标是最终在 Stellar 一侧持有至少 250 枚。桥接流程由源链存入、relayer 生成 Ed25519 attestation、目标链完成释放三部分组成；每次完成会收取金额的八分之一作为手续费。

官方仓库没有文字题解，但给出了完整的 `ex.py`、`ex2.py`。脚本利用的核心是：完成跨链操作时，调用者可以自行指定 `fee_recipient`，而桥接证明没有把该字段绑定进被认证的消息。

## 解题过程

先分别跟踪两个方向的桥接状态：

- Sui 到 Stellar：Sui 侧锁定资产，Stellar 侧根据证明铸造或释放资产；
- Stellar 到 Sui：Stellar 侧产生证明，Sui 侧完成转账；
- 两侧完成函数都会计算 `fee = amount / 8`，并向调用参数中的 `fee_recipient` 支付手续费。

两份合约都只验证 `recipient || amount || message_id`。以 Sui Move 合约为例：

```move
verify_attestation(
    &bridge.relayer_pubkey,
    recipient,
    amount,
    &message_id,
    &attestation,
);

let fee = amount / FEE_DENOMINATOR;
transfer::public_transfer(fee_coin, fee_recipient);
```

`fee_recipient` 参与资产流向，却没有进入签名消息。因此，观察到诚实桥接请求后，可以复用同一份有效 attestation，抢先调用目标链完成函数，并把手续费接收者改为玩家地址。

### 抢跑诚实桥接手续费

官方脚本同时监控两类公开状态：

- Stellar 的 pending transaction 中，来源为 Sui、函数为 `complete_from_sui` 的待处理调用；
- Stellar 的 `deposit_to_sui` 事件和 Sui attestation 接口中，来源为 Stellar、函数为 `complete_from_stellar` 的证明。

发现非玩家的诚实桥接后，脚本立即使用原始 `recipient`、`amount`、`message_id` 和签名，只把 `fee_recipient` 换成玩家。目标链以 `message_id` 防重放，所以攻击者与正常 relayer 竞争的是“谁先完成”，不是伪造签名。

Sui 方向的时间窗口更短。`PreparedSuiCrash` 会预先准备一对共享特殊 coin/object 的冲突交易：一笔转移该对象，另一笔又把它作为 gas/object 使用，并发提交使题目内的 Sui RPC 暂时不可用。这个动作只用于延缓正常 relayer、扩大抢跑窗口；它不是余额下溢，也不是手续费漏洞本身。RPC 恢复后，脚本再用自己的 `fee_recipient` 完成 pending attestation。

脚本持续截取八分之一手续费，直到

$$
\text{Stellar 余额}+\text{Sui 余额}\ge 250.
$$

### 绕过 Stellar 单地址接收上限

积累够总余额后，还要把资产集中到 Stellar。Soroban 合约限制：

```rust
if received + amount > STELLAR_RECEIVE_CAP + buffered {
    return Err(BridgeError::ReceiveCapExceeded);
}
```

其中 `STELLAR_RECEIVE_CAP = 100`。但当接收者的 Stellar trustline 未授权时，`complete_from_sui` 不会把消息标记为已处理，而是调用 `buffer_release()`，同时增加该地址的 `ReleaseBuffer` 和该消息的 `BufferedMessage`。

官方脚本据此执行：

1. 先把玩家在 Stellar 上的余额桥到 Sui，使 Stellar 余额归零。
2. 将 Sui 总额按每块不超过 100 拆分并逐块 `deposit_to_stellar`，取得多份 attestation。
3. 删除玩家的 Stellar trustline，使接收者暂时未授权。
4. 对每份 attestation 调用一次 `complete_from_sui`；转账失败，但金额被计入 release buffer，消息仍未标记 processed。
5. 重新建立 trustline，再重放相同 attestation。扩大的 buffer 使接收上限检查通过。
6. 同时把玩家设为 `recipient` 和 `fee_recipient`，所以每块的“到账金额 + 手续费”都回到玩家，最终 Stellar 余额达到完整总额。

这里不能把“证明有效”等同于“整笔目标链交易的所有参数都已认证”。安全条件应写成：

$$
\operatorname{Verify}\bigl(
\text{amount},
\text{source},
\text{destination},
\text{fee\_recipient},
\text{nonce},
\text{chain\_id}
\bigr)=\text{true}.
$$

题目实现遗漏了其中的 `fee_recipient`。

最终以 Stellar 余额不少于 250、各 attestation 已处理为验证条件，服务才返回 flag。

最终得到：

```text
SEKAI{super-duper-stellar-master-3a9bb1}
```

## 方法总结

这是跨链消息字段绑定不完整导致的费用劫持。审计桥时应逐字段比较“源链事件”“证明正文”和“目标链调用参数”，任何未被签名或未被证明约束、却会影响资产流向的字段都可成为攻击点。抢跑只是放大手段，根因仍是认证消息与执行参数不一致。
