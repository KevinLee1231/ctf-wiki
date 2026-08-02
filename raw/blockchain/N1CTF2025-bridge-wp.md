# N1CTF 2025 Bridge

## 题目简述

题目实现了一个 Solana/Anchor 跨链桥。管理员只为 wSOL 注册 `MintConfig`，配置的单次跨链上限为 $10^{12}$；服务器执行选手程序和一次 `bridge` 指令后，不检查链上资产，而是重新解析交易日志中的 `BridgeEvent`。只有找到满足以下条件的事件才返回 flag：

```rust
event.user == player
    && event.mint == wsol_mint
    && event.amount > 1_000_000_000_000
    && event.target_chain == 42
```

正常 `bridge` 又要求 `amount <= mint_config.max_bridge_amount`，所以无法靠合法存入直接越过上限。预期解法组合了账户约束缺失、Token-2022 Transfer Hook 和 Anchor 0.31.1 客户端事件解析器的日志栈错乱。

## 解题过程

首先检查 `Bridge` 账户结构。`mint` 与 `mint_config` 都由用户提供，但后者既没有 PDA seeds 约束，也没有 `has_one = mint`：

```rust
pub struct Bridge<'info> {
    #[account(mut)]
    pub user: Signer<'info>,
    pub config: Account<'info, BridgeConfig>,
    pub mint: InterfaceAccount<'info, Mint>,
    pub mint_config: Account<'info, MintConfig>,
    // ...
}
```

处理函数只读取 `mint_config.enabled` 和 `max_bridge_amount`，从未验证

```text
mint_config.mint == mint.key()
```

也没有重新推导 `[b"mint-config", mint.key()]`。因此，可以把管理员为 wSOL 创建的有效配置与攻击者自己创建的任意 Token-2022 mint 配对。这个缺陷本身仍不能生成超限事件，但它允许选择带 Transfer Hook 扩展的恶意 mint：跨链桥调用 `invoke_transfer_checked` 时，Token-2022 会在桥程序的 CPI 调用栈内执行选手程序。

服务器使用的日志解析逻辑来自 Anchor 0.31.1。它维护一个仅凭日志文本更新的 `Execution` 栈，并用下面的表达式识别程序返回：

```rust
let re = Regex::new(r"^Program (.*) success*$").unwrap();
```

这个正则没有约束程序 ID，也没有要求标准日志的完整形态。恶意程序执行

```rust
msg!("success");
```

时，运行时产生的 `Program log: success` 也能匹配该正则，解析器便错误地弹出一个调用栈帧。再配合一次或多次 self-CPI，真实的 `Program <id> invoke [n]`、`Program <id> success` 与伪造的 `Program log: success` 会让解析栈比真实 CPI 栈多弹出一层。最终，恶意 Transfer Hook 仍在运行，离线解析器却认为当前执行程序已经回到了题目 bridge 程序。

此时从恶意程序调用 `sol_log_data` 输出伪造事件字节。`handle_program_log` 只检查当前的解析栈顶是否等于 bridge 程序 ID，再检查数据是否以 `BridgeEvent` discriminator 开头，并不知道这条 `Program data:` 实际由哪个程序发出。因此伪造内容可设置为：

```rust
BridgeEvent {
    user: player,
    mint: wsol_mint,
    amount: 1_000_000_000_001,
    dest: Pubkey::default(),
    target_chain: 42,
}
```

完整攻击顺序如下：

1. 创建一个 Token-2022 mint，启用 Transfer Hook，并把 hook 程序设为选手程序。
2. 为玩家铸造足够数量的该 token，准备用户 ATA、escrow ATA 及 hook 所需的额外账户。
3. 调用题目 `bridge` 时，传入攻击者 mint，却复用 wSOL 的 `mint_config`；使用不超过 $10^{12}$ 的真实转账数量，保证指令本身成功。
4. Transfer Hook 被触发后，通过 self-CPI 和 `msg!("success")` 让 Anchor 的离线执行栈提前退回 bridge 上下文。
5. 输出带正确 discriminator 和 Borsh 字段顺序的伪造 `BridgeEvent`。
6. 真实桥事件仍会出现，但其 mint 和数量不满足条件；服务器遍历日志时会命中伪造事件并返回 flag。

上游 [Anchor PR #3657](https://github.com/solana-foundation/anchor/pull/3657) 将问题描述为事件解析 panic，并改用更严格的正则；本题进一步利用同一个根因完成事件伪造。外链提供的是补丁来源，关键的旧正则、栈错乱方式和本题事件字段均已在正文中给出。

## 方法总结

这道题同时跨越链上与链下信任边界。链上缺少 `mint_config` 与 `mint` 的绑定，使攻击者能引入 Transfer Hook；链下 keeper 又把可由程序影响的文本日志当作可信事件源，且 Anchor 0.31.1 的宽松正则允许伪造栈返回。单独修复任意一层都能阻断利用：账户侧应约束 PDA 并检查 `has_one = mint`，事件侧则应使用严格的运行时日志语法、校验调用深度，并避免仅凭可伪造文本决定大额跨链状态。
