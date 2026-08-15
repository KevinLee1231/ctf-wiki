# BSidesAlgiers2025 - Quantum Interference

## 题目简述

题目是一个“量子密钥分发+密文加密”框架：

- Alice 生成大量 EPR 对（`Alice.pairs_data`）并公开 `frames`；
- Bob 选择量子基与测量并返回 `frames` 匹配结果与 `sifting_strings`；
- 双方在一致规则下衍生共享密钥 `shared_secret`；
- `QuantumCryptoSystem.encrypt` 用 `sha256(shared_secret)` 作为 AES-ECB 密钥加密 flag（输出在 `out.py`）。

`out.py` 仅包含公开信息（`frames`、`sifting_strings`、`encrypted_flag`），服务端并不直接把基信息给出，核心是从公开量子误差信息反推密钥。

## 解题过程

### 1. 读取公开数据

`dist/out.py` 的三个字段可抽象成下面的完整结构（变量名表示对应的实际长列表或十六进制字符串）：

```python
public_data = {
    "frames": frame_pairs,
    "sifting_strings": published_sifting_strings,
    "encrypted_flag": encrypted_flag_hex,
}
```

其中 `sifting_string` 格式是 `"{s_bits},{m_bits}"`，`m_bits` 来自 `Bob` 的测量比特，`s_bits` 是 sifting bit。

### 2. 通过 sifting 值恢复帧关系，反推各端基底

`solution.solve_bases` 逐帧构造关系：

- 已知一帧 `u,v` 的 `m_bits = r_u r_v`。
- 若 `s_bits` 与 `r_u,r_v` 匹配“同基”规则，则该帧为 `relation=0`，否则为 `relation=1`。

具体代码规则可读为：

```python
possible_same = (s_bits == f"{r1^r2}0") or (s_bits == f"0{r1^r2}")
possible_diff = (s_bits == f"{r1}{r2}") or (s_bits == f"{r2}{r1}")
```

将 `relation` 在 `frames` 上构建图约束后，用 BFS 指派每个 qubit 的基底标签（X/Z 0/1）。

这一步是关键，因为它把“未公开基底”从 sifting 关系里恢复出来。

### 3. 恢复可用帧的共享密钥字节

对每一帧，在候选态集合中筛选 `USABLE_FRAMES`。官方辅助表只有四种可用有序组合，solver 还会检查反向顺序：

```python
USABLE_FRAMES = {
    ("1x,0z", "1x,1z"): "f2",
    ("0x,1z", "1x,1z"): "f3",
    ("1x,1z", "0x,1z"): "f4",
    ("1x,1z", "1x,0z"): "f6",
}

KEY_DERIVATION = {
    f"{s:02b},{b:02b}": {
        f"{m:02b}": str((((s ^ b) + 1) & 4 >> 2 ^ 1) ^ (m >> 2))
        for m in range(4)
    }
    for s in range(4)
    for b in range(4)
    if (s ^ b) in (0, 3)
}
```

再复现 Alice 的丢弃规则：

- 若 `calc_s_bits` 为 `"01"` 或 `"10"`，该帧丢弃；
- `KEY_DERIVATION` 根据 sifting 字符串和测量结果给出该帧应追加的一个密钥 bit。

`solution.py` 的做法是直接按公开 `sifting_strings`、重构基底、`USABLE_FRAMES` 与 `KEY_DERIVATION` 累加得到候选密钥 `reconstructed_key`。

### 4. 解密验证并处理长度歧义

`encrypted_flag` 按 `AES.new(sha256(key),AES.MODE_ECB)` 解密。脚本做了一层长度回退尝试：

```python
for i in range(len(reconstructed_key)):
    candidate_key_str = reconstructed_key if i == 0 else reconstructed_key[:-i]
```

逐个候选去解密，只要出现 `b"shellmates{"` 即认为成功（脚本内使用该特征判断）。

### 5. 可执行链条

```bash
python3 solution/solution.py
```

前提：`out.py` 必须来自同一实例的完整导出（否则约束图会失配）。

本地使用仓库内 `solution/out.py` 复核时，脚本恢复出 59 对相对基底，并在把候选密钥末尾裁掉 107 bit 后解出：

`shellmates{fRAme_BaSED_Qkd_c4n_Be_unrelIABl3!_h0Pe_y0U_DID_n0T_R3v3R$3_th3_qUAntum_c1rcU1T$_LOL}`

官方输出末尾仍带一组 `0x10` 填充字符，因此结果应截取到第一个闭合花括号；不能把终端中的不可见填充误写进 flag。

## 方法总结

- 核心技巧：将 `sifting_string` 与测量结果转成帧间逻辑约束，先还原每个 qubit 的基底关系，再复用协议本身的 key derivation 逻辑，最终离线枚举密钥裁剪长度。
- 识别信号：当 QKD/BB84 题中只泄漏 sift 信息且服务端允许“对抗/纠错策略”时，公共规则常常足够还原密钥生成过程。
- 复用要点：`USABLE_FRAMES`、`VALID_SS`、`KEY_DERIVATION` 这类状态表是最关键约束，必须完整写入 WP 以保证脱离附件后仍可复现。
