---
type: family
tags: [crypto, family, block-cipher, aes, cbc, ctr, gcm, mac, oracle, key-derivation]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/aes-modes-mac-and-oracles.md
  - ../raw/ai-ml/SU_easyLLMWP.md
updated: 2026-07-06
---

# 分组密码模式、MAC 与 Oracle 误用技巧族

## 作用边界

本页是对称加密误用 family，负责判断题目是否落在 block mode、nonce/IV/counter、MAC/tag、padding 或可查询 oracle 的组合问题上。

如果核心证据是 RSA/ECC/格/代数参数，不应停留在本页；如果核心证据是文件格式、Web parser 或逆向实现细节，本页只接收已经恢复出的加密边界和可复验输入输出。

## 识别信号

- 题目给出可查询加密/解密/校验接口，或给出 nonce、IV、tag、MAC、错误信息和多组密文。
- 同一 key 下出现重复 nonce/IV/counter，或 IV、nonce、counter 可控、固定、可预测。
- 修改密文后响应差异稳定：padding error、MAC error、JSON parse error、权限变化、明文格式变化。
- 认证与加密边界不清：encrypt-then-MAC / MAC-then-encrypt / raw hash / CRC / 线性 MAC 混用。
- 明文结构已知或可猜：flag 前缀、JSON、PNG/PDF header、固定字段、block boundary。

## 最小证据

- 至少收集两组同 key 下的输入输出，能判断 mode、block size、nonce/IV/counter/tag 的位置。
- 有一个最小可重放请求或脚本，能稳定区分“合法 / 非法 / 格式错误 / padding 错误 / MAC 错误”。
- 明确攻击目标：恢复明文、伪造密文、伪造 tag、恢复 key stream，或绕过认证。
- 如果走 oracle，必须量化 oracle 信号：状态码、错误文本、时间、长度、业务状态或返回对象。

## 分流流程

1. 切分协议字段：`nonce/iv | ciphertext | tag/mac | aad | payload`，不要把整包直接丢给工具。
2. 先用 block size、重复块、长度变化、错误差异判断模式和认证顺序。
3. 根据证据选择最小攻击：padding oracle、CTR/GCM nonce reuse、CBC bitflip、ECB 结构泄漏、MAC 长度扩展或线性伪造。
4. 写最小脚本验证一个字节、一个块或一个字段能否被控制。
5. 扩展到完整明文恢复或目标字段伪造；最后用服务端响应或本地复现做正向验证。

## 路线分流

| 变体 | 优先证据 | 下一跳页面 | 失败后 pivot |
|---|---|---|---|
| CBC padding oracle | padding/MAC/parse 错误可区分，block size 稳定 | [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) | 若错误不可区分，改查 timing、长度或业务状态 oracle。 |
| CBC bitflip / block boundary | 前一块可控，明文有固定字段 | [classical-xor-and-substitution-ciphers.md](classical-xor-and-substitution-ciphers.md) | 若有认证 tag，先判断 MAC 顺序或签名绕过。 |
| CTR/GCM nonce reuse | nonce/counter 重复，同 key 多密文 | [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md) | 如果只有 tag 可查，转 GHASH/forbidden attack 或协议 oracle。 |
| ECB 图像/结构泄漏 | 重复明文块对应重复密文块 | [file-signatures-and-flag-artifact-hunting.md](file-signatures-and-flag-artifact-hunting.md) | 如果块不重复，检查压缩、随机 padding 或分组边界。 |
| 线性 MAC / CRC 伪造 | tag 满足 XOR/GF(2)/CRC 可组合关系 | [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) | 若非线性，回到长度扩展、keyed hash 或签名实现。 |
| LLM / password generator 输出派生 key | `key_derivation`、模型/prompt/temperature 或输出格式公开，且多组密文可收集 | [llm-attacks.md](llm-attacks.md) | 若输出空间不集中或模型不可复现，回查实现侧 key 泄露、弱随机种子或已知明文 oracle。 |
| PDF / HashClash chosen-prefix | 需要构造两个同 hash 但语义不同文件 | [crypto-tooling.md](crypto-tooling.md) | 若服务端二次解析文件，联合 Web/parser differential 页面。 |

## 常见陷阱

- 看到 AES 就直接猜 CBC；应先用字段长度、重复块和错误差异确认模式。
- 把“解密失败”和“业务失败”混在一起；oracle 的价值来自可稳定区分的最小信号。
- GCM nonce reuse 不等于只 XOR 明文；伪造 tag 还要处理 GHASH 关系。
- 只在本地复现 crypto，不复现服务端封包、编码、base64/urlencode 和 JSON 规范化。
- 忽略格式已知明文；flag 前缀、文件头和 JSON key 常常足够恢复 keystream。


## 关联技巧

- [classical-xor-and-substitution-ciphers.md](classical-xor-and-substitution-ciphers.md)
- [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md)
- [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md)
- [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md)
- [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md)
- [file-signatures-and-flag-artifact-hunting.md](file-signatures-and-flag-artifact-hunting.md)
- [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)
- [embedded-python-pyd-custom-aes.md](embedded-python-pyd-custom-aes.md)
- [llm-attacks.md](llm-attacks.md)
- [crypto-tooling.md](crypto-tooling.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [NCTF2026-encryption-wp](../raw/crypto/NCTF2026-encryption-wp.md) | pwn 后 dump `libcipher.so`，发现魔改 AES 的 S-box 被线性化；把整块加密建成 GF(2) 仿射变换矩阵再求逆。 |
| [SU_AESWP](../raw/crypto/SU_AESWP.md) | AES 服务允许分别更新 seed/key，导致当前 S-box 与旧 round keys 不同步；先把 S-box 压成常值恢复 `K10`，再用 probe 值域指纹恢复置换。 |
| [Bugku-EasyVT-wp](../raw/reverse/Bugku-EasyVT-wp.md) | `EasyVT.sys` 模拟 VT-x，驱动 VM-exit handler 只是调度壳；核心校验是 TEA 变体和 RC4，优先静态恢复 handler switch。 |
| [Bugku-JustRe-wp](../raw/reverse/Bugku-JustRe-wp.md) | Flag 分两段：前半段是 DWORD 加法/低字节 XOR 约束，后半段是固定 3DES-ECB 密文和 24 字节 key。 |
| [Bugku-week1_re4-wp](../raw/reverse/Bugku-week1_re4-wp.md) | 识别 TEA 常量 `0x9e3779b9`、4 个 32-bit key 和 32 轮 Feistel；按反编译的 key 下标反向解密。 |
| [Bugku-week4_re2-wp](../raw/reverse/Bugku-week4_re2-wp.md) | AES-128-ECB 校验，key 为 `00..0f`，输入 42 字节但比较 48 字节说明最后 block 零填充。 |
| [HGAME2026-marionette-wp](../raw/reverse/HGAME2026-marionette-wp.md) | 父进程用 `ptrace` 调度子进程 `int3; ret` block；hook 记录 RIP trace 后，还原输入差分和 AES-NI 校验。 |
| [LilacCTF2026-c-plus-plus-plus-plus-wp](../raw/reverse/LilacCTF2026-c-plus-plus-plus-plus-wp.md) | C# Native AOT 中 `XEngine` 是 Twofish-like 16 轮 Feistel；先按 RS/MDS、40 个 round key 和 whitening 恢复固定 key/IV。 |
| [SU_flumelWP](../raw/reverse/SU_flumelWP.md) | Flutter/Dart 输入先经 `Rc4Warp`，再由新版 `libjunk.so` 验证 Hermes bundle 并派生 AES-CBC key/IV；旧 placeholder 会误导。 |
| [SU_protocolWP](../raw/reverse/SU_protocolWP.md) | HTTP 路由很薄，body 先 hex 再进私有协议帧；区分格式错、比较失败和 block 变换后再反推 payload。 |

## 原始资料

- [aes-modes-mac-and-oracles.md](../raw/crypto/aes-modes-mac-and-oracles.md)
- [SU_easyLLMWP.md](../raw/ai-ml/SU_easyLLMWP.md)
