# kyber decryption oracle

## 题目简述

服务生成 Kyber512 的 IND-CPA 密钥对，公布 800 字节公钥，并允许至多 56 次提交任意 768 字节密文。它不会直接返回解出的 32 字节消息，而是返回 `sha256(m)`；查询结束后用 $\operatorname{SHA512}(\mathrm{sk})$ 与 flag 异或。因此目标是恢复 Kyber secret key，而非猜测 flag。

Kyber512 有 $k=2$ 个 256 系数秘密多项式，秘密系数很小。裸 IND-CPA 解密函数对攻击者密文没有 CCA 拒绝机制，故可构造密文让消息位成为某个秘密系数的阈值测试；SHA-256 只把这个 bit oracle 包装为一个小范围预像问题，并未消除泄露。

## 解题过程

官方 `kyber_util.py` 固定参数为 $q=3329$、$k=2$、环 $R=\mathbb F_q[X]/(X^{256}+1)$。对第 `block_idx` 个秘密多项式构造：

```python
Pb = (R^2).zero()
Pb[block_idx] = round(q / 16)
c2 = R(h_vec)
ct = polyvec_compress(Pb) + poly_pack(c2)
```

在解密端，所选坐标中会出现由 $h_i$ 和该秘密系数乘以 $q/16$ 决定的量；随后 Kyber 的 `poly_tomsg` 阈值把它变为消息 bit。`BRT_TABLE` 是针对 $s_i\in\{-3,-2,-1,0,1,2,3\}$ 的二叉判定树：状态 `STATE_1` 到终态 `FIN_1`--`FIN_7` 的路径由这些消息 bit 选择，终态值就是恢复的系数。

一次查询同时让 26 个位置处于活动状态。虽然服务只给 SHA-256，官方 `precomp.py` 预先计算“仅这 26 个 bit 可变、其余为零”的消息哈希前 4 字节到 bitset 的映射；solver 以返回哈希的同一前缀反查该批 26 个响应 bit。余下很少的未决 bit 用完整 SHA-256 做离线枚举。求解器的非自适应排程对两个多项式分别推进判定树：第一段共 54 个查询，最后两个 hash 查询处理残余状态，刚好等于 `MAX_QUERIES = 56`。

判定树不能直接覆盖的少量系数由公钥补全。公开键满足小误差 LWE 型关系 $t=A s+e\pmod q$。solver 将 $t,A$ 逆 NTT 后变成常规矩阵，把已知的 $s_i$ 坐标以 `integratePerfectHint` 加入 `LWELattice`，以 BKZ（最大块大小 40）恢复余下短秘密向量。最后按 Kyber 的 NTT 与序列化格式重建 `sk`，并执行：

```python
s_bytes = polyvec_to_bytes(s)   # s 已按 Kyber 格式完成 NTT
key = sha512(s_bytes).digest()
flag = xor(flag_enc, key[:len(flag_enc)])
```

源码提供了完整 attack 脚本，但其预计算与格约化明显耗时；本文仅根据该静态实现归纳，未在本地运行。仓库配置记录的验证值为：

```text
DUCTF{decryption_oracle_is_too_powerful!!}
```

## 方法总结

Kyber KEM 的 CCA 安全来自封装/重加密验证等完整流程；把底层 CPA-PKE 的原始解密 oracle 直接暴露，即使只返回哈希，也会产生 chosen-ciphertext key-recovery attack。哈希不是访问控制：当明文空间被攻击者限制到若干可枚举消息位时，预计算或离线预像就能恢复 oracle 结果。生产实现应只暴露带隐式拒绝的 KEM decapsulation 接口，且避免依据解密结果给出可区分反馈。
