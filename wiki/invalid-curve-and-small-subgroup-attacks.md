---
type: technique
tags: [crypto, ecc, invalid-curve, small-subgroup, crt]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/ecc-dlp-and-signature-attacks.md
  - ../raw/crypto/ACTF2022-cryptonote-wp.md
  - ../raw/crypto/NCTF2026-yqs-wp.md
updated: 2026-07-28
---

# Invalid-Curve and Small-Subgroup Attacks

## 适用场景

协议让秘密标量乘以攻击者提供的椭圆曲线点或群元素，却没有验证曲线归属、点阶、cofactor 或子群成员关系；可用多个小阶查询恢复秘密标量的模因子，再经 CRT 合并。

## 识别信号

- 接口直接接收 `(x, y)`、序列化公钥或群元素，并执行静态私钥乘法。
- 服务只检查坐标格式，不检查点是否在目标曲线或正确子群。
- 响应、MAC、解密结果或错误状态可区分共享秘密候选。
- 可在 twist、无效曲线或目标群的小子群中构造已知阶元素。

## 最小证据

- 证明输入元素实际进入秘密标量运算，而不是先被库拒绝或 cofactor clearing。
- 找到元素的准确阶，并能从响应确定秘密标量模该阶的 residue。
- CRT 累积模数覆盖私钥搜索范围，候选可由公开公钥或完整协议正向验证。

## 解法骨架

1. 复现点解码与验证路径，确认缺失的是 on-curve、subgroup 还是 cofactor 检查。
2. 选择两两互素的小阶点/元素，逐轮提交并枚举对应小子群响应。
3. 记录每轮得到的 `d mod r_i`，排除符号或响应碰撞歧义。
4. 用 CRT 合并 residue；模数不足时继续收集，不要猜高位。
5. 以 `dG`、共享秘密或协议输出验证最终标量。

## 关键变体

| 变体 | 元素来源 | 验证重点 |
|---|---|---|
| Invalid curve | 改变曲线常数但复用相同加法公式 | 点不在目标曲线，阶在替代曲线上可控。 |
| Quadratic twist | 使用 twist 上的小阶点 | 实现接受编码且未做严格曲线检查。 |
| Small subgroup | 目标群本身含小因子 | 缺少子群检查或 cofactor clearing。 |

## 常见陷阱

- 只找到小阶点，却没有可观察的响应判别 residue。
- 把点阶、曲线基域素数和主子群阶混用。
- CRT 模数乘积尚未覆盖私钥空间就认定唯一解。
- 库已经做严格点验证，继续构造无效点只会得到统一错误。

## 关联技巧

- [signature-nonce-reuse-and-partial-leakage.md](signature-nonce-reuse-and-partial-leakage.md)
- [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md)
- [algebraic-polynomial-and-modular-root-reconstruction.md](algebraic-polynomial-and-modular-root-reconstruction.md)

## 原始资料

- [ecc-dlp-and-signature-attacks.md](../raw/crypto/ecc-dlp-and-signature-attacks.md)
- [ACTF2022-cryptonote-wp](../raw/crypto/ACTF2022-cryptonote-wp.md)
- [NCTF2026-yqs-wp](../raw/crypto/NCTF2026-yqs-wp.md)
