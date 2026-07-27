---
type: family
tags: [crypto, family, block-cipher, aes, cbc, ctr, gcm, mac, oracle, key-derivation]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/aes-modes-mac-and-oracles.md
  - ../raw/ai-ml/SUCTF2026-easyLLMWP.md
updated: 2026-07-27
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

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| padding/error/timing 反馈可建立自适应查询谓词 | [adaptive-oracle-response-modeling.md](adaptive-oracle-response-modeling.md) |
| MAC/hash 构造、拼接边界或长度扩展导致认证失效 | [hash-mac-construction-and-length-extension.md](hash-mac-construction-and-length-extension.md) |
| 故障密文差分直接泄露对称密钥或轮状态 | [symmetric-cipher-differential-fault-analysis.md](symmetric-cipher-differential-fault-analysis.md) |

## 常见陷阱

- 看到 AES 就直接猜 CBC；应先用字段长度、重复块和错误差异确认模式。
- 把“解密失败”和“业务失败”混在一起；oracle 的价值来自可稳定区分的最小信号。
- GCM nonce reuse 不等于只 XOR 明文；伪造 tag 还要处理 GHASH 关系。
- 只在本地复现 crypto，不复现服务端封包、编码、base64/urlencode 和 JSON 规范化。
- 忽略格式已知明文；flag 前缀、文件头和 JSON key 常常足够恢复 keystream。


## 关联技巧

- [classical-xor-and-substitution-ciphers.md](classical-xor-and-substitution-ciphers.md)
- [reduced-round-spn-integral-attacks.md](reduced-round-spn-integral-attacks.md)
- [symmetric-cipher-differential-fault-analysis.md](symmetric-cipher-differential-fault-analysis.md)
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
| [SUCTF2026-AESWP](../raw/crypto/SUCTF2026-AESWP.md) | AES 服务允许分别更新 seed/key，导致当前 S-box 与旧 round keys 不同步；先把 S-box 压成常值恢复 `K10`，再用 probe 值域指纹恢复置换。 |
| [西湖论剑2023-EasyVT-wp](../raw/reverse/西湖论剑2023-EasyVT-wp.md) | `EasyVT.sys` 模拟 VT-x，驱动 VM-exit handler 只是调度壳；核心校验是 TEA 变体和 RC4，优先静态恢复 handler switch。 |
| [强网杯2019-JustRe-wp](../raw/reverse/强网杯2019-JustRe-wp.md) | Flag 分两段：前半段是 DWORD 加法/低字节 XOR 约束，后半段是固定 3DES-ECB 密文和 24 字节 key。 |
| [HGAME2026-marionette-wp](../raw/reverse/HGAME2026-marionette-wp.md) | 父进程用 `ptrace` 调度子进程 `int3; ret` block；hook 记录 RIP trace 后，还原输入差分和 AES-NI 校验。 |
| [LilacCTF2026-c-plus-plus-plus-plus-wp](../raw/reverse/LilacCTF2026-c-plus-plus-plus-plus-wp.md) | C# Native AOT 中 `XEngine` 是 Twofish-like 16 轮 Feistel；先按 RS/MDS、40 个 round key 和 whitening 恢复固定 key/IV。 |
| [SUCTF2026-flumelWP](../raw/reverse/SUCTF2026-flumelWP.md) | Flutter/Dart 输入先经 `Rc4Warp`，再由新版 `libjunk.so` 验证 Hermes bundle 并派生 AES-CBC key/IV；旧 placeholder 会误导。 |
| [SUCTF2026-protocolWP](../raw/reverse/SUCTF2026-protocolWP.md) | HTTP 路由很薄，body 先 hex 再进私有协议帧；区分格式错、比较失败和 block 变换后再反推 payload。 |
| [0xGame2024-week4-DES-wp](../raw/crypto/0xGame2024-week4-DES-wp.md) | 面对“简化 DES”应先按源码重建轮结构，不能直接套标准 DES 工具。本题最致命的设计是只有两轮且复用同一子密钥，使每轮的 $F$ 输入输出都能由明密文确定；逆 P 置换后，48 位搜索又自然分解为 8 次 6 位枚举，整体工作量很小。 |
| [ACTF2022-retros-wp](../raw/crypto/ACTF2022-retros-wp.md) | 这道题把密码利用与受限程序设计串在一起。第一阶段的核心不是破解 AES，而是利用“填充错误”和“填充正确后继续执行”两条可区分路径，逐字节恢复分组解密中间值；第二阶段则要把双重循环、条件交换和跳转复用压缩到 31 字节以内。 |
| [D3CTF2024-Sym-signin-wp](../raw/crypto/D3CTF2024-Sym-signin-wp.md) | 利用短周期轮密钥调度构造滑动攻击，把完整 8192 轮验证缩减为 4 轮关系检查；轮数极大但轮函数相同、轮密钥序列周期很短，并提供多组有结构的明密文时，应优先检查 slide pair。 |
| [MoeCTF2023-feistel-1-2-wp](../raw/crypto/MoeCTF2023-feistel-1-2-wp.md) | Feistel 网络的可逆性不要求轮函数可逆；解密只依赖交换结构和逆序轮密钥。隐藏密钥版本则利用了模 $2^n$ 运算的低位封闭性：已知一块明文后，可把 64 位搜索拆成逐位提升，并用分支集合处理暂时不唯一的低位解。 |
| [UMDCTF2018-whitepaper-crypto-wp](../raw/crypto/UMDCTF2018-whitepaper-crypto-wp.md) | AES 密钥扩展不是单向函数，知道任意完整的 128 位轮密钥就能反推出主密钥。处理 PDF 密码题时还应逐字符核对十六进制数据；一个字符转写错误就会使整块解密结果失去可读性。 |
| [UMDCTF2023-aes-tr-wp](../raw/crypto/UMDCTF2023-aes-tr-wp.md) | 利用确定性分组密码保留输入相等关系，构造消息使错误的“计数器异或明文后再加密”方案产生可观察碰撞；自定义模式若直接计算 $E_k(\text{nonce/counter}\oplus m)$，应立即检查不同分组的 AES 输入是否可被控制为相同。 |
| [UMDCTF2023-cbc-mac-2-wp](../raw/crypto/UMDCTF2023-cbc-mac-2-wp.md) | 利用 CBC 状态可控拼接，通过两个查询标签的异或把伪造链同步到另一条已知链的中间状态；长度虽被追加，但仍作为普通末尾分组进入同一个 CBC-MAC，且攻击者能查询多种长度时，不能直接认为长度扩展问题已经消失。 |
| [UMDCTF2024-haes-2-wp](../raw/crypto/UMDCTF2024-haes-2-wp.md) | 用三个选择明文建立差分，依据公开线性层传播活动字节，再用 S 盒 DDT 反推出末轮密钥；低轮 SPN、末轮省略线性混合、选择明文数量很少但允许精确控制差分时，应考虑按列或按字节的差分密钥恢复。 |
| [UMDCTF2024-triple-des-wp](../raw/crypto/UMDCTF2024-triple-des-wp.md) | 分析多层 CBC 的差分传播，确认最外层 IV 的异或改动会穿过三次解密，最终直接作用于单个原始明文块；无论嵌套多少层，只要解密端暴露最终填充是否合法，并且某个前置块可控，就要检查是否仍保留 CBC 可塑性。 |
| [UMDCTF2025-bedrock-block-wp](../raw/crypto/UMDCTF2025-bedrock-block-wp.md) | 逆掉线性 XOR-rotate 层，再用选择明文的相邻差分观察模加法进位链，从统计偏差恢复轮密钥；少轮 ARX 结构、同一密钥复用、可批量选择明文，以及“加法后只有线性旋转异或扩散”都提示可以分层剥离。 |
| [UMDCTF2025-obsidian-block-wp](../raw/crypto/UMDCTF2025-obsidian-block-wp.md) | 把循环旋转转成模 $2^w-1$ 的乘法，把模 $2^w$ 加法的非线性压缩成每轮一个可枚举进位位；重复轮密钥、仅由加法和固定旋转组成、轮数足以枚举进位状态但不足以提供安全性。 |
| [WMCTF2020-game-wp](../raw/crypto/WMCTF2020-game-wp.md) | BEAST 类攻击的识别信号是：CBC 模式、攻击者能控制明文前缀、IV 或前一块密文可预测、目标 secret 被拼接进同一次加密。关键不是“解 AES”，而是利用 CBC 的异或输入性质，把 CBC oracle 转化成可控单块加密 oracle。只要能把未知字节对齐到块尾，并知道同块前 15 字节，就能用 256 次以内的枚举恢复 1 字节 secret。 |
| [WMCTF2020-idiot-box-wp](../raw/crypto/WMCTF2020-idiot-box-wp.md) | 差分分析题的第一步是对 S 盒求差分分布表，而不是直接爆破密钥。本题的异常点在于 6 进 4 出非双射 S 盒存在单 S 盒两轮迭代高概率差分，使五轮特征概率达到可利用范围。利用时按 S 盒分段选择明文对、过滤满足差分传播的密文对、统计第六轮子密钥候选，最后把 8 个 6-bit 分段组合并逆密钥扩展验证明文中是否出现 `WMCTF`。 |
| [WMCTF2022-ocococb-wp](../raw/crypto/WMCTF2022-ocococb-wp.md) | 本题核心是 OCB nonce 复用下的 offset 复原和认证绕过。OCB 的每个块会用由 $E(\text{nonce})$ 派生出的 offset 做异或；nonce 可控且可复用时，可以通过构造明文/密文关系恢复这些 offset，并伪造后续需要的中间块。 |

## 原始资料

- [aes-modes-mac-and-oracles.md](../raw/crypto/aes-modes-mac-and-oracles.md)
- [SUCTF2026-easyLLMWP.md](../raw/ai-ml/SUCTF2026-easyLLMWP.md)
