---
type: technique
tags: [crypto, web, oracle, adaptive-query, side-channel]
skills: [ctf-crypto, ctf-web]
raw:
  - ../raw/crypto/hash-protocol-and-oracle-attacks.md
  - ../raw/crypto/prng-z3-lcg-and-timing-attacks.md
  - ../raw/web/SUCTF2026-sqliWP.md
  - ../raw/crypto/UMDCTF2026-no-brainrot-allowed-wp.md
updated: 2026-07-28
---

# Adaptive Oracle Response Modeling

## 适用场景

目标不会直接返回秘密，但对攻击者可控输入泄露比较、padding、格式、长度、错误类型、执行时间或成功/失败位，可转化为可重复的查询决策过程。

## 识别信号

- 输入只改变一个受控变量，响应类别或 timing 随秘密关系变化。
- 反馈可用于排除候选、收缩区间或逐字节恢复状态。
- 查询存在噪声、限流或多种错误来源，需要先做分类。

## 最小证据

- 至少两组已知关系样本能稳定落入不同响应类别。
- 给出“响应类别 -> 数学谓词”的明确映射。
- 测量重试次数、误判率和查询预算。

## 解法骨架

1. 将网络异常、协议错误和真实 oracle 响应分层记录。
2. 用控制变量实验确定每一类反馈对应的秘密谓词。
3. 选择二分、逐字节、区间维护、候选过滤或统计检验循环。
4. 每轮保存状态和原始证据；恢复后使用独立验证器确认。

## 关键变体

- Deterministic oracle：状态码、错误文本或布尔值可直接分类。
- Timing oracle：需要随机化请求顺序、重复采样和稳健统计量。
- Stateful oracle：查询会改变 nonce、计数器或会话，必须建模状态迁移。

## 常见陷阱

- 只观察一次响应就建立谓词。
- 把 CDN、缓存、连接复用或限流噪声当作 timing 泄露。
- 查询循环不可断点恢复，前序状态丢失后无法复算。

## 关联技巧

- [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md)
- [rsa-padding-and-interval-oracle-attacks.md](rsa-padding-and-interval-oracle-attacks.md)
- [prng-z3-lcg-and-timing-attacks.md](prng-z3-lcg-and-timing-attacks.md)

## 原始资料

- [hash-protocol-and-oracle-attacks.md](../raw/crypto/hash-protocol-and-oracle-attacks.md)
- [prng-z3-lcg-and-timing-attacks.md](../raw/crypto/prng-z3-lcg-and-timing-attacks.md)
- [SUCTF2026-sqliWP](../raw/web/SUCTF2026-sqliWP.md)
- [UMDCTF2026-no-brainrot-allowed-wp](../raw/crypto/UMDCTF2026-no-brainrot-allowed-wp.md)
