# launchpad WP

## 题目简述

这是一道 Solana/Anchor 题。程序围绕 faucet、claim request 和 token vault 设计了多组 PDA 和账户约束，但其中若干约束只校验了表面关系，没有把“当前 faucet、mint、receipt、claim request 必须属于同一组状态”绑定起来。

关键业务接口包括 `CreateClaimRequest`、`ClaimBatch`、`CreateFaucet` 和 `CloseFaucet`。漏洞分别来自 `faucet_mint` 未绑定当前 faucet、批量领取时未确认 claim request 属于当前 faucet、receipt owner 可被 SPL Token `set_authority` 修改，以及关闭 faucet 时未验证传入 mint 与 faucet 记录一致。

## 解题过程

先看 `CreateClaimRequest`。它接收 `faucet` 和 `faucet_mint`，但没有强制 `faucet_mint == faucet.mint`，导致 claim request 中记录的 receipt 可以来自另一个 mint。

再看 `ClaimBatch`。它通过 remaining accounts 批量处理 `(claim_request, receipt)`，但只验证 claim request 自身 PDA 和 receipt owner，没有检查该 `claim_request_account.faucet` 必须等于当前传入的 `faucet`。因此可以拿“自己 faucet 产生的 claim request”，去当前上下文中的其它 faucet vault 领币。

此外，`ClaimBatch` 中的 `is_on_curve(receipt_ta.owner)` 检查也不可靠，因为 SPL TokenAccount 的 owner 可以通过 `set_authority` 转移。利用时先用用户身份正常 `simple_claim` 一次，再用任意 PDA 创建用户和 claim request，把该 PDA 的 receipt token account 通过 `set_authority` 转给用户，最后调用 `claim_batch`，即可绕过“每个用户只能领取一次”的限制并凑够创建新 faucet 所需费用。

拿到足够费用后有两条路线：

1. 创建一个高 `per_user_amount` 的新 faucet。
2. 调用 `CreateClaimRequest` 时把 `faucet_mint` 换成管理员 faucet 的 mint。
3. 调用 `ClaimBatch` 时传入管理员 faucet 和 vault，但 remaining accounts 中放入前一步构造的 claim request。
4. `ClaimBatch` 使用当前 faucet vault 转账，却按 claim request 中的 amount 结算，从而抽空目标 vault。

第二条路线利用 `CloseFaucet`。`CloseFaucet` 没有把 `faucet_mint` 与 `faucet.mint` 强绑定，只要求 `refund_account == faucet.refund_account`。可以先创建一个 refund account 受控的 faucet，随后关闭这个 TokenAccount，再用同一个地址重新初始化为另一个 mint 的账户。地址没变，但 mint 已变，最终在 `CloseFaucet` 中把其它 mint 的 vault 转到这个 refund account。

## 方法总结

本题核心不是 PDA 公式写错，而是账户约束缺少跨字段一致性校验。Solana/Anchor 审计时不能只看单个 `#[account(...)]` 是否存在，还要确认 mint、faucet、vault、receipt、claim request 是否绑定到同一个业务对象。`set_authority` 这类 SPL Token 原语也会让“owner 是谁”变成可变状态，不能拿它当不可伪造身份。
