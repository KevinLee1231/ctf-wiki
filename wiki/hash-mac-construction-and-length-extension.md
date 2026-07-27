---
type: technique
tags: [crypto, hash, mac, length-extension, construction]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/hash-protocol-and-oracle-attacks.md
updated: 2026-07-27
---

# Hash/MAC Construction and Length Extension

## 适用场景

协议把 hash 误当 MAC、拼接字段无长度绑定，或验证端与生成端对消息边界理解不同，使攻击者能扩展、重排或碰撞被认证内容。

## 识别信号

- 签名形如 `Hash(secret || message)`、`Hash(message || secret)` 或自制 keyed hash。
- 字段通过裸拼接、弱分隔符或不同编码组合后再哈希。
- 服务允许提交修改后的消息和 tag，并明确反馈验签结果。

## 最小证据

- 确认底层 hash 的 Merkle-Damgard 结构、digest 长度和 padding 规则。
- 明确 secret 位于前缀还是后缀，以及可枚举长度范围。
- 保存生成端与验证端使用的精确字节串。

## 解法骨架

1. 还原消息序列化和 MAC 构造，不先假设 HMAC。
2. 前缀 secret 且为可扩展 hash 时，枚举 secret 长度并注入 glue padding。
3. 裸字段拼接时构造边界歧义或编码等价消息。
4. 用服务端验签 oracle 验证，成功后再组合业务字段。

## 关键变体

- Length extension：适用于 MD5/SHA-1/SHA-256 等前缀 secret 构造，不适用于标准 HMAC。
- Ambiguous concatenation：字段无长度绑定时可重划边界。
- Parser differential：哈希字节视图和业务解析视图不一致。

## 常见陷阱

- 看到 hash 就盲试长度扩展，未确认 secret 位置和构造。
- 对字符串而非真实编码字节计算 padding。
- 忽略 URL/JSON/Unicode 正规化对验签输入的影响。

## 关联技巧

- [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md)
- [adaptive-oracle-response-modeling.md](adaptive-oracle-response-modeling.md)
- [json-duplicate-key-hmac-parser-differential.md](json-duplicate-key-hmac-parser-differential.md)

## 原始资料

- [hash-protocol-and-oracle-attacks.md](../raw/crypto/hash-protocol-and-oracle-attacks.md)
