# iihash

## 题目简述

服务在启动时随机生成一个未知 64 位 seed，使用 `XXH3_128` 计算哈希，并提供任意次查询；查询和最终 payload 都必须严格长于 256 字节。目标是提交一段消息，使其 128 位摘要的字节串恰好等于：

```text
Give me the flag
```

官方 `solve.py` 同时完成 seed 恢复和长输入预映像构造。

## 解题过程

XXH3 的长输入模式不是把消息简单送入一个理想压缩函数，而是使用由 seed 调整的 192 字节固定 secret、8 个 64 位累加器、64 字节 stripe、合并阶段和可逆 avalanche。effective secret 按 64 位小端字交替派生：

```python
for i in range(12):
    secret_u64[2*i]   = (default_u64[2*i]   + seed) % 2**64
    secret_u64[2*i+1] = (default_u64[2*i+1] - seed) % 2**64
```

### 恢复 seed

加减 seed 时，低 32 位是否进位只会在有限个边界处变化。官方脚本用默认 secret 的低 32 位生成这些边界，把 $2^{32}$ 个候选划成若干区间；每个区间取一个代表 `seed_lower32`，即可确定前 15 个 secret 半字的进位修正。

随后构造矩阵：

```python
M = matrix(ZZ, [
    [1, -1] * 7 + [1],
    secrets_u32_adjusted[:15],
])
M = M.right_kernel().matrix().LLL()
```

短核向量给出若干输入字之间的线性改变量。`try_find_collision()` 用 meet-in-the-middle 找到两组小输入，使候选 secret 下的异或差分相消；把两组长度大于 256 字节的消息交给哈希 oracle：

- 若摘要不同，当前进位区间或高位候选错误；
- 若多组独立关系都碰撞，则保留候选。

脚本先枚举 seed 高 32 位的低 4 位，再逐位扩展到完整高 32 位；已知高半后，又用四个可控 32 位输入构造第二类碰撞，逐位恢复低 32 位。合并两半得到完整 seed，再按上式重建 effective secret。

### 构造目标预映像

目标固定为 16 字节 `b"Give me the flag"`。官方脚本选择一条 1024 字节长消息，因为它稳定进入 XXH3 long-input 路径，并按以下顺序逆推：

1. `avalanche_inv()` 逐位逆掉 `x ^= x >> 32`、乘法 `PRIME_MX1` 和 `x ^= x >> 37`；乘法通过模 $2^{64}$ 的逆元还原。
2. `merge_acc_inv()` 从目标的高、低 64 位反推出 8 个 accumulator lane。三个 `mul128_fold64` 项的和用 LLL、Babai nearest-plane 和 meet-in-the-middle 求解。
3. `acc_512_128_inv()` 按逆序减去最后 15 条 stripe 对累加器的加法和 $32\times32$ 乘积贡献。
4. 前两条 stripe 共 128 字节不直接逆乘法；脚本先放入与 secret 相关的基准字节，再把目标 accumulator 与基准 accumulator 的差分别写入各 lane 的低/高 32 位。
5. 保持剩余 896 字节为固定填充，得到完整 1024 字节 payload。

验证条件是：

```python
assert len(payload) == 1024
assert xxh3_128_digest(payload, seed=seed) == b"Give me the flag"
```

提交构造出的 payload 后得到：

```text
SEKAI{Im4g1n3_U5iNg_LLL_!n_d1ff3rEn7IAL_Cryp7aN@lySis}
```

## 方法总结

本题的关键不是暴力搜索 128 位摘要，而是利用非密码学哈希为速度设计的可逆、线性和低扩散结构。遇到带 seed 的高速哈希时，应先分清短输入与长输入分支，再检查 secret 派生、累加器更新和 avalanche 中哪些步骤可逆或能通过差分消去。
