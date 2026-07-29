# Super Secure Store

## 题目简述

题目是一个 Solana 商店程序。服务初始化规范的 `CONFIG`、`ITEM` 和多个 `PRODUCT` PDA，给用户 1 SOL；最终要求用户余额严格大于 95 SOL，并且商品 5、7 的 `current_owner` 都是用户。

程序同时存在两个相互配合的问题：

- `Init` 接受调用者提供的 bump，只用 `create_program_address` 验证 PDA，没有强制使用 `find_program_address` 返回的规范 bump；
- `Buy` 与 `Withdraw` 只检查 `Config`、`Item` 的 owner 和数据类别，没有确认它们就是规范 PDA。

这允许用户创建第二套合法但非规范的 Config/Item，再把伪 Config 与真实资金池 Item 混用。

## 解题过程

### 创建非规范 PDA

服务已经用 `find_program_address([b"CONFIG"])` 和 `find_program_address([b"ITEM"])` 创建了规范账户。Solana 的规范 bump 只是从 255 向下找到的第一个有效值；同一组种子通常还存在其他能通过 `create_program_address` 的 bump。

在解题程序中遍历 0 到 254，排除服务给出的规范地址，分别寻找另一组有效 PDA：

```rust
fn alternative_pda(seed: &[u8], program: &Pubkey, canonical: Pubkey)
    -> (Pubkey, u8)
{
    for bump in 0u8..=254 {
        if let Ok(addr) =
            Pubkey::create_program_address(&[seed, &[bump]], program)
        {
            if addr != canonical {
                return (addr, bump);
            }
        }
    }
    panic!("no alternative PDA");
}
```

然后再次调用 `Init`，签名者使用用户自己，并传入非规范的 Config/Item 地址和 bump。程序会创建：

```text
fake_config.admin = user
fake_config.total_balance = 10
fake_item.category = Item
```

这些账户确实由目标程序拥有，因此能通过后续类型和 owner 检查。

### 用零金额购买制造下溢

`Buy` 会验证商品是对应 ID 的规范 Product，但不会验证传入的 Config 和 Item 地址。转账金额也允许为 0，结尾却无条件执行：

```rust
config_data.total_balance =
    config_data.total_balance.checked_add(lamports).unwrap() - 1;
```

使用 `fake_config`、`fake_item` 和真实商品账户调用 11 次 `Buy(amount=0)`。前 10 次令 `total_balance` 从 10 降到 0，第 11 次在 release 构建中发生无符号下溢，变为 `u64::MAX`。购买过程中要至少各传一次真实的商品 5 和商品 7，因为每次调用都会执行：

```rust
product_data.current_owner = *user.key;
```

一种简单安排是对商品 5 调用 10 次，对商品 7 调用 1 次。

### 从真实资金池提款

`Withdraw` 同样没有校验 Config 与 Item 是否属于同一套规范 PDA。传入：

- 管理员是自己的 `fake_config`；
- 保存真实资金的规范 `item_addr`；
- 提款金额 `95_000_000_000` lamports。

伪 Config 的 `total_balance` 已经下溢为极大值，管理员检查也因 `admin=user` 通过；真实 Item 则在服务初始化时收到了管理员购买商品所支付的 99 SOL，足够完成实际扣款。用户原有 1 SOL，提款后变为 96 SOL，满足严格大于 95 SOL 的条件。商品 5、7 的所有者也已在零金额购买时改成用户，服务遂返回 flag。

## 方法总结

本题的根因不是单独一个算术下溢，而是账户身份验证不完整。PDA 约束必须同时固定种子、程序 ID 和规范 bump；仅验证“由本程序拥有且能反序列化成某类型”会允许伪造平行状态。跨指令审计时还应检查不同账户是否属于同一状态集合。这里先用非规范 PDA 伪造管理员与账面余额，再把伪账本和真实资金池拼接，最后利用零金额购买触发下溢并满足商品所有权条件。
