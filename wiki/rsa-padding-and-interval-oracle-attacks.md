---
type: technique
tags: [crypto, rsa, padding-oracle, interval-oracle, adaptive-query]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/rsa-specialized-structures-and-oracles.md
  - ../raw/crypto/UMDCTF2026-no-brainrot-allowed-wp.md
updated: 2026-07-28
---

# RSA Padding and Interval Oracle Attacks

## 适用场景

服务对乘法变换后的 RSA 密文暴露 PKCS#1/OAEP 格式、明文区间、奇偶性、长度或 timing 差异，可通过自适应查询逐步收缩明文候选区间。

## 识别信号

- 可提交任意密文或其乘法变换，并得到可分类响应。
- 响应与解密后明文的首字节、padding、范围或奇偶性相关。
- 同一密文重复查询的分类稳定，或 timing 差异可经统计放大。

## 最小证据

- 保存原始请求、响应分类和重试统计。
- 证明密文乘以 `s^e mod n` 后对应明文乘以 `s mod n`。
- 明确 oracle 判定对应的数学区间，而不是只凭状态码猜测。

## 解法骨架

1. 建立可重放查询器，统一网络异常、限流和噪声处理。
2. 用已知合法/非法样本校准 oracle 类别。
3. 按 Bleichenbacher、Manger 或 parity 模型维护区间集合并选择下一查询因子。
4. 区间收敛到单值后重新加密验证，再移除真实 padding。

## 关键变体

- PKCS#1 v1.5：合法前缀把明文限制在 `[2B,3B)`。
- OAEP/Manger：oracle 常反映明文是否跨过某个字节长度边界。
- Parity/timing：每次只泄露一位或统计差异，需更多查询与抗噪。

## 常见陷阱

- 将传输失败、限流和真实 oracle 响应混为一类。
- 区间端点取整错误导致候选被提前排除。
- 得到 padded block 后忘记按原协议解析。

## 关联技巧

- [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)
- [adaptive-oracle-response-modeling.md](adaptive-oracle-response-modeling.md)
- [rsa-factor-relation-and-partial-key-recovery.md](rsa-factor-relation-and-partial-key-recovery.md)

## 原始资料

- [rsa-specialized-structures-and-oracles.md](../raw/crypto/rsa-specialized-structures-and-oracles.md)
- [UMDCTF2026-no-brainrot-allowed-wp](../raw/crypto/UMDCTF2026-no-brainrot-allowed-wp.md)
