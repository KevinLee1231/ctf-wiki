---
type: technique
tags: [crypto, fault-analysis, aes, speck, dfa, technique]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/UMDCTF2022-dfa-aes-wp.md
  - ../raw/crypto/UMDCTF2022-dfa-speck-1-wp.md
  - ../raw/crypto/UMDCTF2022-dfa-speck-2-wp.md
updated: 2026-07-27
---

# Symmetric-Cipher Differential Fault Analysis

## 适用场景

题目给出正确密文和在已知或受限轮次、位置注入故障后产生的密文。通过比较正确/故障执行的差分传播，可以逐字节或逐位恢复末轮轮密钥，再逆 key schedule 或逐轮剥离。

## 识别信号

- 数据按正确密文与多组 fault ciphertext 配对。
- 故障发生在末轮前、某个固定轮或某个比特位置。
- AES 类 SPN 的末轮缺少线性混合，或 ARX 密码的加法进位链可由差分长度观察。
- 故障编号、分组或位置与密文差分有稳定对应关系。

## 最小证据

- 固定正确/故障样本配对、故障时刻、故障宽度和可能位置。
- 完整确认分组字节顺序、字内端序、状态布局和轮函数。
- 在一个样本上手算或脚本验证差分传播模型。
- 明确目标是末轮轮密钥还是主密钥，以及 key schedule 是否可逆。

## 解法骨架

1. 对每组样本计算输出差分，并按故障位置/传播形状归组。
2. 枚举局部末轮密钥，逆最后一层非线性，检查输入差分是否满足故障模型。
3. AES 类按字节/列相交候选；ARX 类利用进位链长度递推密钥位。
4. 恢复末轮密钥后逆 key schedule，或逆一轮把下一组样本化归相同故障模型。
5. 用主密钥完整解密，并以明文格式、可打印性和重新加密共同消除二义性。

## 关键变体

- AES 末轮前单字节/单比特故障可将候选限制为局部逆 S 盒差分。
- Speck 等 ARX 密码通过模加法进位传播泄露密钥位，最高位可能保留二义性。
- 多轮故障需要从最后一轮向前逐层剥离，不能一次联立所有轮。
- RSA/CRT fault 的数学结构不同，应转到 [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)。

## 常见陷阱

- 将文件中的两个字拼接顺序、整数显示端序和明文字节序混为一层。
- 只按故障编号分组，没有检查真实差分形状。
- 得到末轮密钥就直接用于解密，忽略主密钥派生。
- 过早丢弃最高位或端序的少量候选，没有用完整加解密验证。

## 关联技巧

- [block-mode-misuse-family.md](block-mode-misuse-family.md)
- [reduced-round-spn-integral-attacks.md](reduced-round-spn-integral-attacks.md)
- [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)
- [crypto-tooling.md](crypto-tooling.md)

## 原始资料

- [UMDCTF2022-dfa-aes-wp.md](../raw/crypto/UMDCTF2022-dfa-aes-wp.md)
- [UMDCTF2022-dfa-speck-1-wp.md](../raw/crypto/UMDCTF2022-dfa-speck-1-wp.md)
- [UMDCTF2022-dfa-speck-2-wp.md](../raw/crypto/UMDCTF2022-dfa-speck-2-wp.md)
