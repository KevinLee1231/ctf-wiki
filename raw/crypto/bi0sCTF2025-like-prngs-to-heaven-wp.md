# Like PRNGS to Heaven

## 题目简述

这题是典型“多源脆弱随机性 + ECDSA 部分 nonce 泄漏 + Lattice 还原密钥”的复合题。
可复现入口由三类事件构成：

- `get_encrypted_flag`
- `perform_deadcoin`
- `call_the_signer`
- `level_restart`

`admin/README.md` 及源码都指向两条主线：`perform_deadcoin` 可泄露 MT 原始输出，签名又泄露部分 k 约束，最终 `d` 可被 lattice 求出并解密 flag。

[出题人题解](https://blog.bi0s.in/2025/06/13/Crypto/Elliptic-Curves/LikePRNGStoHeaven-bi0sCTF2025/)补充确认了 `biased ECDSA + Ripley seeded MT + EHNP/HNP` 方向；外链中的决定性步骤已经写入本文，链接只用于核对原始说明。

## 解题过程

`chall.py` 的签名系统里：

1. 私钥 `d` 由 `CORE` 经过简单 LCG + `supreme_RNG`（平方数中间位）拼接，非标准 CSPRNG；
2. `nonce k` 由随机位拼块、`sec_real_bits` 的 MT 采样及 `partial_noncense_gen` 组成；
3. 签名返回 `r,s` 外，还返回：
   - `nonce_gen_consts` 两组 `[_and, equation]`
   - `heat_gen` 两个循环计数（`MT` 在该签名前已消耗的长度）。

`perform_deadcoin` 在 `deadcoin_verification` 中会返回 `blood = Max_Sec.get_num()`，并通过一次动作码校验把 `feedbacker_parry` 公开给用户。脚本里取 3 次就是为了恢复 `R_MT19937_32bit` 的状态。

`admin/exploit/solve.sage` 的核心步骤：

1. 先执行 375 次 `level_restart`，把内部状态推进到官方脚本预期位置；随后执行 3 次 `perform_deadcoin`，取得 3 条带索引的 MT 观测；
2. 用自实现 Z3 方程 `BreakerRipley32` 反推 seed：
   - 用 51 步 LCG 变换把 seed 映射到 MT 初始化；
   - 建模 `MT_twist` 与 `tamper_state`；
3. 用恢复的 seed 重构 `R_MT19937_32bit`，预测未来 `k` 关键片段；
4. 用 `nonce_gen_consts` 解出两段部分随机项：

$$
term \oplus ((term\ll16)\ \&\ \text{mask}) = equation
$$

5. 按 `Kbar` 组装近似 nonce，调用 `ecdsa_key_disclosure(...)`（lbc toolkit）做隐藏数恢复；
6. 用 `d` 解密 `get_encrypted_flag` 的 AES-CBC 密文。

```python
def partial_nonce_breaker(_and, equation):
    term = BitVec('term', 32)
    res = term ^ ((term << 16) & _and)
    ...
    if S.check() == sat:
        return model[term].as_long()
```

```sage
Kbar = 2^224*(b1 >> 24 & 0xFF) + 2^192*(b1 >> 16 & 0xFF) + \
        2^160*k1 + 2^152*(b1 >> 8 & 0xFF) + 2^75*(b1 & 0xFF) + \
        2^33*(b2 >> 24 & 0xFFF) + k2
```

`ecdsa_key_disclosure` 得到主私钥后，按 `sha256(str(d))[:16]` 解密得到明文 flag。

```text
bi0sCTF{p4rry_7h15_y0u_f1l7hy_w4r_m4ch1n3}
```

## 方法总结

- 关键不是单一签名公式，而是把三类信息统一到一条链：
  - MT 种子（Ripley 恢复）
  - k 的低位偏置约束（位级方程）
  - ECDSA HNP 约束（格攻击）
- 官方脚本在 `solve.sage` 已给出完整模板；复盘时仍要逐项核对 MT 输出、`nonce_gen_consts`、`heat_gen` 与五条签名，不能把外部 HNP 工具包当作不透明的一键解密器。
- 外部信息（如 `bi0s` 官方博客）可作为辅助，但正文已完整保留机制，不依赖外链也可复现。
