---
type: technique
tags: [forensics, technique]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/blockchain-and-transaction-forensics.md
updated: 2026-05-21
---

# Blockchain and Transaction Forensics

## 适用场景

题目给出 TXID、地址、区块高度、钱包导出、交易截图或链上金额流转，需要从公开链数据中恢复收款方、找零链、隐藏 memo、交易时间线或最终资金落点。它属于取证而不是密码学：核心证据来自链上交易图和金额行为。

## 识别信号

- 题面出现 Bitcoin、Ethereum、wallet、TXID、address、mempool、block explorer、UTXO、gas、transfer 等线索。
- 已知一个起点地址或交易，但 flag 依赖继续追踪后续输出。
- 多笔交易中存在一个较大输出和一个较小“找零/剥离”输出。
- 输出金额呈现 round amount、手续费模式或 peel chain 规律。

## 最小证据

- 至少确认一个有效 TXID 或地址，并能用区块浏览器/API 查到输入输出。
- 能解释每个候选输出的角色：收款、找零、手续费或继续转移。
- 如果采用 peel-chain 判断，应给出“为什么跟随较大输出或特定金额输出”的证据，而不是只凭直觉跳地址。

## 解法骨架

1. 用 `mempool.space/api/tx/<TXID>` 或区块浏览器拉取交易 JSON。
2. 建表记录 `txid -> inputs -> outputs -> value -> address`，避免在浏览器里手工迷路。
3. 对 UTXO 模型优先跟踪金额更大的输出；round amount 常是剥离给目标的一侧，找零金额通常继续进入下一跳。
4. 每跳记录区块时间、高度、金额变化和地址复用。
5. 最后检查地址标签、OP_RETURN、交易备注、NFT/代币 metadata 或最终输出金额是否编码 flag。

## 关键变体

| 变体 | 复用重点 |
|---|---|
| Bitcoin peel chain | 从起点 TXID 开始，优先跟踪较大输出；round amount 或固定小额输出常是剥离支付。 |
| 地址聚类 | 多输入交易常暗示这些输入地址由同一实体控制，可作为聚类证据。 |
| OP_RETURN / memo | 不要只看金额；交易输出脚本和 metadata 可能直接存 flag。 |
| 跨链/代币 | EVM 题要看 internal transaction、event log 和 token transfer，而不仅是主币转账。 |

## 常见陷阱

- 把“最大输出”规则机械套用；遇到 mixer、交易所聚合或 CoinJoin 时必须重新判断。
- 忽略手续费导致金额差异解释错误。
- 只看网页 UI，不保存 API JSON，后续无法复现路径。
- 把链上追踪误判为 crypto 数学题，浪费时间找私钥或签名漏洞。

## 关联技巧

- [3d-printing.md](3d-printing.md)
- [audio-frequency-and-archive-stego.md](audio-frequency-and-archive-stego.md)
- [cross-domain-forensics-technique-map.md](cross-domain-forensics-technique-map.md)
- [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)
- [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md)

## 原始资料

- [blockchain-and-transaction-forensics.md](../raw/forensics/blockchain-and-transaction-forensics.md)
