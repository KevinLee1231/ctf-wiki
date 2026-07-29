# GFT

## 题目简述

GFT 是运行在 Solana CTF 框架中的链上抽卡程序。服务端给用户账户 2000 lamports、金库 50000 lamports，并加载参赛者上传的求解程序；执行一次求解指令后，若用户余额大于 40000 就返回 flag。

虽然赛事仓库把它放在 Pwn 目录，决定性漏洞是 Solana 账户类型验证与 Borsh 布局混淆，因此归入 Blockchain 更恰当。

## 解题过程

程序定义了两种账户结构：

```rust
pub struct UserAccount {
    pub primos: u64,
    pub characters: Vec<u8>,
    pub owner: Pubkey,
}

pub struct Character {
    pub stars: u64,
    pub name: String,
    pub id: u8,
    pub owner: Pubkey,
}
```

`buy_primos` 会检查传入账户归挑战程序所有，却没有验证它一定是 `UserAccount`：

```rust
let mut useraccount =
    UserAccount::deserialize(
        &mut &useraccount_info.data.borrow()[..],
    )?;

assert_eq!(useraccount_info.owner, program);
useraccount.primos += amount;
useraccount.serialize(
    &mut &mut useraccount_info.data.borrow_mut()[..],
)?;
```

Borsh 不写入类型标签，而两个结构的开头又完全同构：

```text
UserAccount: u64 primos | u32 vec_len | bytes... | Pubkey
Character:   u64 stars  | u32 name_len | bytes... | u8 id | Pubkey
```

选择角色 Dori 尤其合适：其 `stars=4`、名字长度也是 4。把 Dori 的 Character PDA 传给 `buy_primos` 时：

- `stars` 被解释为 `primos`；
- 字符串 `"Dori"` 被解释为四个角色编号；
- `id=4` 与后续 31 个 owner 字节暂时组成 `UserAccount.owner`；
- 序列化写回的长度不变，未覆盖的最后一个 owner 字节仍保留，因此它之后仍能按 `Character` 正确解析。

完整链上调用顺序如下。

先为账户名 `727` 计算 PDA：

```text
["ACCOUNT", user_pubkey, "727"]
```

再计算 Dori 与金库 PDA：

```text
["CHARACTER", useraccount_pubkey, 0x04]
["VAULT"]
```

创建用户账户后购买 800 primos。四星角色单价为：

$$
\begin{aligned}
4\times\text{BASE\_PRICE}
&=4\times200 \\
&=800.
\end{aligned}
$$

所以恰好可以买下 `character_id=4` 的 Dori。然后再次调用 `BuyPrimos`，但把第一个账户替换成 Dori Character PDA，并令：

```rust
amount = 255 - 4;
```

结果是角色的 `stars` 从 4 增加到 255。最后正常调用 `SellAccount` 并提交该角色。出售价格按角色账户里未经上限校验的星级计算：

$$
\frac{255\times200\times80}{100}=40800.
$$

初始 2000 lamports 扣除购买 primos 的 800，以及创建两个 PDA 各 10 lamports 后剩 1180；卖出后约为 41980，超过服务端要求的 40000。官方求解程序通过 CPI 顺序完成以上四次调用，得到：

```text
SEKAI{nfts_b4d_s0l4n4_b3tt3r_g3nsh1n_b3st}
```

## 方法总结

Solana 的 owner 校验只能证明账户属于某个程序，不能证明账户内部属于哪种业务类型。若多个 Borsh 结构没有显式 discriminator，且字段前缀碰巧兼容，同一个账户就可能在不同指令中被反序列化成不同类型。本题利用 `u64` 首字段对齐，把充值数量变成角色星级，再让出售公式放大该值。
