---
type: technique
tags: [crypto, spn, aes, integral, square-attack, technique]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/0xGame2024-week4-AES-wp.md
  - ../raw/crypto/0xGame2025-week4-蓉凤-wp.md
  - ../raw/crypto/UMDCTF2024-haes-wp.md
updated: 2026-07-27
---

# Reduced-Round SPN Integral Attacks

## 适用场景

AES 或自定义 SPN 的轮数明显减少，服务允许获得大量选择明文密文对，末轮省略线性混合或能按小组逆推。攻击利用一组结构化明文在中间状态的 active、constant、balanced 或 distinct 性质，将末轮密钥搜索拆成字节或小组枚举。

## 识别信号

- 轮数约 3 至 6 轮，远低于标准参数。
- 可查询完整 delta set 或多个只改变某些位置的明文集合。
- 末轮没有 MixColumns/线性层，或状态置换使逆 S 盒可局部计算。
- 分组/字节运行在小域或自定义有限域，单位置候选空间较小。

## 最小证据

- 准确还原状态布局、字节顺序、S 盒、线性层、轮密钥加法和末轮差异。
- 用小样本追踪一个选择明文集合，确认目标中间位置的平衡或不重复性质。
- 明确每次猜测覆盖一个字节、一个列还是四字节组。
- 准备第二个独立集合或完整加密验证来压低伪候选。

## 解法骨架

1. 选一个或多个输入位置遍历完整域，其余位置固定，生成结构化明文集。
2. 跟踪该集合经过各轮后的 active/balanced/distinct 状态。
3. 对候选末轮子密钥执行 `InvSBox(ciphertext xor key_guess)`。
4. 检查恢复状态在集合上的和、异或和、不重复性或题目特定不变量。
5. 交叉多个集合筛选候选，拼出末轮密钥后逆 key schedule 或重做完整加密验证。

## 关键变体

- 完整 delta 集通常提供平衡性质；部分 delta 集可能只能使用不重复性，需联合枚举多个字节。
- 小域实现可能把多个原始字节映射为同一域元素，恢复域密钥后仍需枚举等价表示。
- 固定轮密钥偏移不一定破坏积分性质，可先代数化简，减少需猜轮数。
- 自定义状态很大时仍可能逐字节恢复，但布局/置换错误会让所有候选失败。

## 常见陷阱

- 对错误密文字节做逆 S 盒，误判积分性质不存在。
- 固定套用“异或和为零”，没有根据实际域和集合追踪不变量。
- 只用一个集合接受大量伪候选，没有做第二集合或完整验证。
- 恢复末轮密钥后直接解密，忽略 key schedule 或原始字节到域元素的多对一映射。

## 关联技巧

- [block-mode-misuse-family.md](block-mode-misuse-family.md)
- [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md)
- [symmetric-cipher-differential-fault-analysis.md](symmetric-cipher-differential-fault-analysis.md)
- [crypto-tooling.md](crypto-tooling.md)

## 原始资料

- [0xGame2024-week4-AES-wp.md](../raw/crypto/0xGame2024-week4-AES-wp.md)
- [0xGame2025-week4-蓉凤-wp.md](../raw/crypto/0xGame2025-week4-蓉凤-wp.md)
- [UMDCTF2024-haes-wp.md](../raw/crypto/UMDCTF2024-haes-wp.md)
