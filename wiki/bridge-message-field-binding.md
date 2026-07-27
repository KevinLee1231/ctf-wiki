---
type: technique
tags: [blockchain, bridge, message-binding, front-running, technique]
skills: [ctf-blockchain]
raw:
  - ../raw/blockchain/SekaiCTF2026-OuterStellar-wp.md
updated: 2026-07-27
---

# Bridge Message Field Binding

## 适用场景

跨链桥、跨域消息或离线证明先认证一份源链事件，再由目标链函数接收额外参数完成铸造、转账、手续费结算或收款。若认证正文没有绑定所有影响资产流向的执行字段，攻击者可以复用合法证明并替换未绑定参数。

## 识别信号

- 目标链调用同时携带 proof/signature 和 recipient、relayer、fee、token、amount 等独立参数。
- verifier 只哈希源链事件的部分字段，执行函数却读取更多 calldata。
- 同一证明可被第三方提交，且 mempool 中能观察到合法交易。
- nonce/claim 防重放存在，但不能阻止第一次提交时替换收款或费用字段。

## 最小证据

- 并排列出源链事件、证明编码、签名/哈希输入和目标链函数参数。
- 标出每个会改变资产流向、金额或权限的字段由哪一层认证。
- 用不变证明替换一个目标字段，确认 verifier 仍通过且执行结果改变。
- 区分根因与放大手段：字段未绑定是根因，抢跑只是取得首次提交权。

## 解法骨架

1. 还原跨链消息的规范编码和签名域，避免只按 ABI 参数名猜测。
2. 构造字段绑定矩阵：每个执行字段是否进入事件、proof、签名和 replay key。
3. 选择未绑定且影响资产的字段，保持证明正文不变生成替代调用。
4. 若有防重放，确保替代交易先于合法提交或使用尚未消费的消息。
5. 比较执行前后余额、claim 状态和事件，证明资产确实流向攻击者。

## 关键变体

- recipient 已绑定但 relayer fee 未绑定时，可劫持手续费而非本金。
- amount/token 未绑定时影响更大，但还需检查目标链是否从事件另行读取。
- 签名绑定字段而 replay key 未绑定时，可能产生跨合约、跨链或跨实例重放。
- 多编码格式并存时，要检查域分隔符、链 ID 和合约地址是否一并绑定。

## 常见陷阱

- 看到抢跑就把问题归为纯 MEV，忽略即使私有提交也存在的字段认证缺口。
- 只比较 Solidity 参数名，没有按真实序列化字节验证签名输入。
- 防重放键存在就认为消息安全，没有检查键与执行字段的覆盖关系。
- 替换参数后只看交易成功，未核对实际余额和费用归属。

## 关联技巧

- [blockchain-smart-contract-exploitation.md](blockchain-smart-contract-exploitation.md)
- [delegatecall-storage-layout-corruption.md](delegatecall-storage-layout-corruption.md)
- [auth-jwt.md](auth-jwt.md)

## 原始资料

- [SekaiCTF2026-OuterStellar-wp.md](../raw/blockchain/SekaiCTF2026-OuterStellar-wp.md)
