# moar-horse-5

## 题目简述

题目部署 Solana 程序：用户用 lamports 购买 horse，再卖回 vault。初始余额只有 2000，服务端要求最终余额超过 100000。`buy` 计算 `HORSE_COST * amount` 时使用 `u64` 且没有检查溢出，可用极小成本记入巨额 horse 数量。

## 解题过程

合约中 `HORSE_COST = 1000`。求解程序选择：

```rust
let amount = (u64::MAX / 1000) + 1;
```

发布构建中乘法按 $2^{64}$ 回绕：

$$
1000\times\left(\left\lfloor\frac{2^{64}-1}{1000}\right\rfloor+1\right)
\equiv384\pmod{2^{64}}.
$$

因此 `buy` 只从用户转走 384 lamports，却执行 `wallet_data.amount += amount`，记录了足以通过后续余额检查的巨额库存。随后调用 `Sell { amount: 100 }`：库存检查通过，vault 向用户转回 $1000\times100=100000$ lamports。最终余额为

$$
2000-384+100000=101616>100000.
$$

利用程序通过 CPI 依次调用目标程序的 Buy 和 Sell 指令；客户端上传编译后的 BPF `.so`，并传入目标程序、用户、horse PDA、wallet PDA 与系统程序账户。服务端检查余额后返回：

```text
tjctf{too_much_horse_769dc05b1cad6497}
```

## 方法总结

- 链上金额与数量运算必须使用 `checked_mul`、`checked_add` 等显式溢出检查，不能依赖构建模式行为。
- 本题的资产凭证 `amount` 与实际付款在同一次溢出计算后失去守恒关系，随后正常 Sell 就能抽走 vault 资金。
- 审计 Solana 题时除 PDA 与 signer 校验外，还要逐项检查金额单位、整数边界和 CPI 前后的状态一致性。
