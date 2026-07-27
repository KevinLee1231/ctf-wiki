---
type: technique
tags: [crypto, homomorphic, malleability, paillier, elgamal, oracle]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/homomorphic-and-exotic-algebra.md
updated: 2026-07-27
---

# Homomorphic Ciphertext Malleability Oracles

## 适用场景

Paillier、ElGamal、RSA 或自定义群密码保留可组合代数运算，服务又对变换后密文执行解密、比较、计分或业务逻辑，使攻击者可通过同态变换构造可验证查询。

## 识别信号

- 密文乘法/幂运算对应明文加法、乘法或标量变换。
- 服务禁止原密文但接受与其代数相关的新密文。
- 解密结果不直接返回，却影响余额、比较、菜单或成功状态。

## 最小证据

- 明确密文群、明文群以及可利用的同态公式。
- 证明变换后密文合法且不会因随机化/范围检查被拒绝。
- 给出服务响应对应的明文谓词。

## 解法骨架

1. 从加密公式推导攻击者可控的明文变换。
2. 构造随机化但关系保持的密文，绕过“禁止原样提交”检查。
3. 将业务响应建模为比较/等值 oracle，按候选或区间自适应查询。
4. 恢复明文后重新加密或用独立协议关系验证。

## 关键变体

- Paillier additive homomorphism：密文相乘对应明文相加。
- ElGamal multiplicative malleability：分量变换对应明文乘法。
- Textbook RSA：乘以 `s^e` 对应明文乘 `s`。

## 常见陷阱

- 只看到“同态”就尝试运算，未找可观察 oracle。
- 忽略明文取模回绕和合法范围检查。
- 服务会消耗一次性状态，却按无状态脚本重复查询。

## 关联技巧

- [homomorphic-and-exotic-algebra.md](homomorphic-and-exotic-algebra.md)
- [adaptive-oracle-response-modeling.md](adaptive-oracle-response-modeling.md)
- [algebraic-polynomial-and-modular-root-reconstruction.md](algebraic-polynomial-and-modular-root-reconstruction.md)

## 原始资料

- [homomorphic-and-exotic-algebra.md](../raw/crypto/homomorphic-and-exotic-algebra.md)
