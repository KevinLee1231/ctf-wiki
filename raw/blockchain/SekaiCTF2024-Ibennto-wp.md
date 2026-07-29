# イベント！ (Ibennto!)

## 题目简述

题目是由 Solang 编译到 Solana 的合约。玩家注册后会得到两个 PDA：

- `LiveBonus`：前两个 `uint64` 分别是 `lastClaimedAt` 和 `currentValue`。
- `EventInfo`：第一个 `uint64` 是 `badges`。

商店初始包含 `Hatsune Miku`、`FLAG` 和 `Live Bonus Drink`。兑换函数接收由调用者提供的账户，但只检查 `eventInfo.owner` 属于题目程序，没有验证该账户确实是 `EventInfo` 类型：

```solidity
EventInfo eventInfoData = getEventInfoAccountData(tx.accounts.eventInfo);
assert(tx.accounts.eventInfo.owner == type(ProjectSEKAI).program_id);
```

同时，函数先取得动态数组元素的 storage pointer，随后覆盖数组元素并 `pop`，最后才通过旧 pointer 读取商品名并发送事件。

## 解题过程

### 账户类型混淆

`claimLiveBonus` 会把当前时间戳写入 `LiveBonus.lastClaimedAt`。该值通常远大于商店价格。调用 `exchange` 时，把同一个 `LiveBonus` PDA 同时放到 `liveBonus` 和 `eventInfo` 账户位置：

```solidity
AccountMeta[4] exchangeMetas = [
    AccountMeta({pubkey: data, is_signer: false, is_writable: true}),
    AccountMeta({pubkey: user, is_signer: true, is_writable: false}),
    AccountMeta({pubkey: liveBonus, is_signer: false, is_writable: true}),
    AccountMeta({pubkey: liveBonus, is_signer: false, is_writable: true})
];
```

由于两种结构的首字段都位于偏移 $0$，`exchange` 会把 `lastClaimedAt` 误读为 `badges`，从而绕过余额不足限制。这不是整数溢出，而是缺少账户 discriminator 或 PDA 身份校验造成的类型混淆。

### 利用失效的 storage pointer

兑换逻辑的顺序是：

```solidity
Item storage item = itemsInShop[index];
// 校验价格并扣除 badges
itemsInShop[index] = itemsInShop[itemsInShop.length - 1];
itemsInShop.pop();
emit Redeemed(item.name, tx.accounts.user.key);
```

先兑换索引 `2` 的 `Live Bonus Drink`，数组缩短为两个元素。随后兑换索引 `0` 的 `Hatsune Miku`。此时函数把最后一个元素 `FLAG` 覆盖到索引 `0`，而 `item` 仍指向索引 `0` 的 storage，所以事件读取到的名称已经变成 `FLAG`。

完整调用顺序为：

```solidity
IProjectSEKAI.register{program_id: target, accounts: regMetas}();
IProjectSEKAI.claimLiveBonus{program_id: target, accounts: claimMetas}();

IProjectSEKAI.exchange{program_id: target, accounts: exchangeMetas}(2);
IProjectSEKAI.exchange{program_id: target, accounts: exchangeMetas}(0);
```

Solang 对外部 Solana 程序调用要求账户严格按声明顺序传入；官方求解脚本还计算了 `LiveBonus`、`EventInfo` PDA，并把 system program、clock sysvar 和题目 program 一并列入账户表。最终第二次 `exchange` 发出名称为 `FLAG` 的 `Redeemed` 事件。

仓库中服务端使用的 flag 为：

```text
SEKAI{mY_h4nd5_10vE_12hy7hm_G4m3s_:3}
```

## 方法总结

- 核心技巧：将同一 program-owned 账户按另一种结构解释，再利用动态数组修改后仍读取 storage pointer 的时序问题。
- 识别信号：Solana 指令接受调用者提供的多个账户，但只检查 owner、不检查 PDA、discriminator、数据长度或账户角色。
- 复用要点：Solana 合约必须同时绑定账户 owner、预期 PDA 和账户类型；Solidity/Solang 中取得 storage reference 后再执行数组覆盖或 `pop`，需要重新判断该引用最终指向的数据。
