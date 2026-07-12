---
type: tooling
tags: [crypto, tooling, tools, environment]
skills: [ctf-crypto]
updated: 2026-07-11
---

# Crypto Tooling

本页记录 `ctf-crypto` 方向的本机工具清单、调用层、路径和适用边界。`SKILL.md` 只保留首轮工具摘要；需要详细路径、环境和专项工具说明时读取本页。

## 工具选择边界

### 入口选择

- 小脚本和对称/哈希实验默认走 `ctf-tools`；数论、格、ECC、Coppersmith 和 Sage 专属能力走 `sage` 环境。
- RsaCtfTool、HashClash 属于专项全路径/独立项目工具，必须按本页路径调用，不要假设全局命令可用。
- RSA 模数首轮先查 FactorDB Web/API；只有当前会话确实暴露 callable 的 FactorDB MCP 时才走 MCP，再决定是否进入 Sage/RsaCtfTool。

### 不应进入 Crypto 工具链的情况

- Base64、hex、URL、ROT 和普通码表继续按 Crypto 表示层处理；压缩/文件容器证据先看 Forensics，图片像素、QR 碎片或隐藏载荷先看 Stego，不要不加判断地套密码分析。
- JWT、JSON parser、签名接口或 TLS transcript 同时出现时，先保留请求/协议证据，再决定 crypto 方程是否真是主障碍。
- Sage/LLL/Z3 只是“能跑”的工具，不是首轮默认答案；没有变量界和可复算方程时先回到 triage family。

### 补工具经验的触发条件

- raw 出现 EdDSA/ECDSA nonce、BLS、aggregate signature，且需要固定工具链复现。
- 协议型题目需要 KDF、AEAD associated data、handshake transcript 的可复算脚手架。
- timing/power/cache 证据足够具体时，工具边界应同时链接 Crypto 与 Forensics。

## 本机工具清单（按使用时机）

### 首轮常用

| 工具 | 为什么放在首轮 |
|---|---|
| FactorDB Web/API；可选 MCP | RSA 模数或可疑合数出现时，先验证“是不是先分解”；MCP 必须 live verify |
| SageMath | 需要数论、格、ECC、DLP 时的主工作台 |
| `pycryptodome` | 快速识别和复现对称加密模式 |
| RsaCtfTool | 在确认是 RSA 后，快速覆盖常见攻击族 |

### 专项按需

- 数论/代数：`gmpy2`、`sympy`
- 约束与格：`z3-solver`、`fpylll`
- 曲线与配对：`py_ecc`
- 哈希类：`hashpumpy`、HashClash
- 自动化与杂项：`ciphey`、`pwntools`、`numpy`
- 外部爆破/分解：`hashcat`、`john`、`yafu`

### 当前未装 / 建议按需补装

- 当前没有明显缺口。Crypto 这套更需要按题型切换工具，而不是继续堆更多默认依赖。

## 失败信号与转向

- FactorDB、gcd、低指数和近似平方根都无结果：先回到 [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md) 重列参数、未知量和可复算方程，不要盲跑 Sage。
- Sage/LLL/Z3 长时间无解：检查变量界、泄露位宽、模数和方程是否对应真实代码；格问题转 [lattice-and-lwe.md](lattice-and-lwe.md)，PRNG 约束转 [prng-z3-lcg-and-timing-attacks.md](prng-z3-lcg-and-timing-attacks.md)。
- RsaCtfTool 没覆盖：判断是否是 specialized prime/oracle/padding/fault，而不是继续换自动工具；转 [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)。
- 证据来自 Web token、JSON parser、签名接口或 PCAP：保留 crypto 方程，但把请求/协议层转到 Web、Forensics 或 Pentest 对应 family。

## 详细清单

### FactorDB Web/API 与可选 MCP

当前会话未配置 callable 的 FactorDB MCP，不能把下面的 MCP 方法当作稳定入口。默认使用 FactorDB Web/API；只有工具清单实际出现相应方法时，才按下表调用。

| 工具 | 功能 | 调用方式 |
|---|---|---|
| `factordb_query` | 查询大整数分解状态和已知因子 | 仅当 MCP callable；传入 `number` 参数 |
| `factordb_query_by_id` | 按数据库 ID 查询分解结果 | 仅当 MCP callable；传入 `id` 参数 |
| `factordb_report_factor` | 向 FactorDB 提交新发现的因子 | 仅当 MCP callable，且提交外部数据前确认；传入 `number` + `factor` |
| `factordb_report_factor_by_id` | 按 ID 向 FactorDB 提交因子 | 仅当 MCP callable，且提交外部数据前确认；传入 `id` + `factor` |

### Python 包（ctf-tools conda）

| 工具 | 版本 | 功能 | 典型用法 |
|---|---|---|---|
| **pycryptodome** | 3.17 | 对称/非对称加密（AES/RSA/ChaCha20） | `from Crypto.Cipher import AES` |
| **gmpy2** | 2.3.0 | 高精度算术（离散对数/模逆/GCD） | `gmpy2.invert(e, phi)` |
| **sympy** | 1.14.0 | 符号数学（因式分解/素数/代数） | `sympy.factorint(n)` |
| **z3-solver** | 4.13.0 | SMT 约束求解（PRNG/序列号/方程组） | `Solver().add(...); s.check(); s.model()` |
| **pwntools** | 4.15.0 | XOR 处理器/序列化/数据操作 | `from pwn import xor; xor(ct, key)` |
| **ciphey** | 5.14.0 | 全自动解密/编码识别（输入密文→自动识别加密方式并解密） | `ciphey -t "ciphertext"` |
| **numpy** | 2.4.4 | 数值计算（矩阵/统计/向量） | `numpy.linalg.solve(A, b)` |
| **fpylll** | 0.6.4 | LLL/BKZ 格基规约，CVP 求解 | `from fpylll import IntegerMatrix, LLL; LLL.reduction(M)` |
| **py_ecc** | 8.0.0 | 椭圆曲线运算（有限域/配对/标量乘） | `from py_ecc.fields import optimized_bls12_381_FQ as FQ` |
| **hashpumpy** | 1.2 | 哈希长度扩展攻击（MD5/SHA-1/SHA-256） | `hashpumpy.hashpump(known_hash, known_data, append_data, key_len)` |

### SageMath（sage conda）

| 工具 | 版本 | 关键功能 | 典型用法 |
|---|---|---|---|
| **SageMath** | 10.7 | 全面符号数学环境 | `conda activate sage && sage script.sage && conda deactivate` |
| | | LLL/BKZ 格基规约 | `M.LLL(); M.BKZ(block_size=20)` |
| | | Coppersmith 小根攻击 | `f.small_roots(X=2^N, beta=0.5)` |
| | | 离散对数（DLP/Pohlig-Hellman） | `discrete_log(h, g)` |
| | | 椭圆曲线群运算 | `E = EllipticCurve(GF(p), [a, b]); E.random_point()` |
| | | Berlekamp-Massey 算法 | `berlekamp_massey([sequence])` |
| | | GF(2) 线性代数 | `M = matrix(GF(2), data); M.solve_right(v)` |
| | | 多项式操作与因式分解 | `factor(poly); gcd(p1, p2)` |
| | | 中国剩余定理（CRT） | `crt([r1, r2], [m1, m2])` |
| | | Jordan 标准型 | `A.jordan_form()` |
| | | p-adic 数 | `Qp(p, prec).log(a)` |

### RsaCtfTool（独立 venv）

| 工具 | 路径 | 功能 | 激活方式 |
|---|---|---|---|
| **RsaCtfTool** | `/home/kali/RsaCtfTool/venv/bin/RsaCtfTool` | RSA 自动攻击套件（Wiener/Hastad/Fermat/Pollard 等 30+ 方法） | `cd /home/kali/RsaCtfTool && source venv/bin/activate && RsaCtfTool ... && deactivate` |

### 系统全局命令（WSL Kali）

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| **hashcat** | `/usr/bin/hashcat` 7.1.2 | GPU 哈希破解（全类型支持） | `hashcat -m 0 -a 3 hash.txt ?l?l?l?l` |
| **john** | `/usr/sbin/john` 1.9.0-jumbo | CPU 哈希破解 / JWT secret 暴力 | `john hash.txt --wordlist=rockyou.txt` |
| **xxd** | `/usr/bin/xxd` | 十六进制转储与还原 | `xxd -p ciphertext \| head` |
| **openssl** | `/usr/bin/openssl` 3.6.1 | 加解密/签名/证书操作 | `openssl rsautl -decrypt -inkey key.pem` |
| **hashid** | `/usr/bin/hashid` 3.1.4 | 哈希类型识别 | `hashid '$2y$10$...'` |
| **yafu** | `/usr/local/bin/yafu` 3.1.2 | 自动化大整数分解（ECM/QS/NFS） | `yafu "factor(@)" -batchfile nums.txt` |

### HashClash（WSL 本地编译）

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| **HashClash cpc.sh** | `/home/kali/hashclash/scripts/cpc.sh` | MD5 选择前缀碰撞生成 | `/home/kali/hashclash/scripts/cpc.sh prefix1 prefix2` |
| **md5_fastcoll** | `/home/kali/hashclash/bin/md5_fastcoll` | MD5 快速碰撞生成 | `/home/kali/hashclash/bin/md5_fastcoll -p prefix -o out1 out2` |

### Windows 本地

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| **CyberChef** | `D:\CTF工具\CyberChef\CyberChef_v10.23.0.html` 或在线 https://gchq.github.io/CyberChef/ | 万能编解码瑞士军刀（300+ 操作，Magic 自动识别编码） | 浏览器打开 html，拖入文件或粘贴文本 |
| **fastcoll** | `D:\CTF工具\fastcoll\fastcoll_v1.0.0.5_exe\fastcoll_v1.0.0.5.exe` | MD5 相同前缀碰撞生成 | `fastcoll_v1.0.0.5.exe -p prefix -o out1 out2` |


---

## 补充速查

- **Python:** `conda activate ctf-tools` 然后 `pip install pycryptodome z3-solver sympy gmpy2`，用完后 `conda deactivate`
- **SageMath:** `conda activate sage` 然后 `sage -python script.py` (required for ECC, Coppersmith, lattice attacks)，用完后 `conda deactivate`
- **RsaCtfTool:** `cd /home/kali/RsaCtfTool && source venv/bin/activate && /home/kali/RsaCtfTool/venv/bin/RsaCtfTool -n <n> -e <e> --uncipher <c> && deactivate` — automated RSA attack suite
- **quipqiup.com:** Automated substitution cipher solver (frequency + word pattern analysis)
