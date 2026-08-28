---
type: tooling
tags: [crypto, tooling, tools, environment]
skills: [ctf-crypto]
---

# Crypto Tooling

本页是 `ctf-crypto` 方向本机工具信息的唯一权威来源，负责维护当前安装状态、版本、路径、环境、完整调用、适用边界和失败处理。`ctf-crypto/SKILL.md` 只说明何时使用工具，`ctf-solve-challenge/SKILL.md` 只负责分流；两者都不复制本页细节。

本页只保留当前真实状态，不维护核验时间、变更历史或旧版本说明。实际环境与本文不一致时，直接修正文中现状。

## 完整调用约定

所有终端命令从 `pwsh` 发起。Conda 工具用固定入口运行；WSL 系统工具和独立项目用其绝对路径：

```pwsh
wsl /home/kali/miniforge3/bin/conda run --no-capture-output -n ctf-tools python /path/to/script.py --example-argument value
wsl /home/kali/miniforge3/bin/conda run --no-capture-output -n sage sage /path/to/script.sage
wsl /usr/bin/openssl version
```

独立 Python venv 是否激活只由该工具自身决定。激活的作用通常只是调整 `PATH`；自动化应直调 venv 绝对入口，需要复合 shell 语义时才在 `conda run` 创建的环境内用最小 `bash -c` 按需 `source`。两种方式都不得依赖终端预先激活的 Conda 或 venv 状态。

## 工具选择边界

### 入口选择

- 小脚本和对称/哈希实验默认走 `ctf-tools`；数论、格、ECC、Coppersmith 和 Sage 专属能力走 `sage` 环境。
- RsaCtfTool、HashClash 属于专项全路径/独立项目工具；先核对本页“当前可用性”，再按本页路径调用，不要假设全局命令可用。
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
| RsaCtfTool | 确认是 RSA、需要覆盖常见攻击族，且本页显示入口可用后再使用 |

### 专项按需

- 数论/代数：`gmpy2`、`sympy`
- 约束与格：`z3-solver`、`fpylll`
- 曲线与配对：`py_ecc`
- 哈希类：`hashpumpy`、HashClash
- 自动化与杂项：`ciphey`、`pwntools`、`numpy`
- 外部爆破/分解：`hashcat`、`john`、`yafu`
- 古典替换密码：在线 `quipqiup`

### 当前可用性

- `ctf-tools` 中本页列出的 Python 包均已安装；SageMath 与系统全局命令可调用，两项 HashClash 入口文件存在且可执行。
- RsaCtfTool 0.1.0 已以 editable 方式安装到项目独立 venv。该 venv 由 `sage` Conda 环境的 Python 3.12 创建且不继承基础 `site-packages`；从 `conda run -n sage` 直调 venv 入口时，RsaCtfTool 依赖留在 venv，子进程 `sage` 由 Sage 环境提供。CLI、依赖一致性、代表性 Python 攻击和真实 Sage 子进程攻击均已验证可用。

## 失败信号与转向

- FactorDB、gcd、低指数和近似平方根都无结果：先回到 [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md) 重列参数、未知量和可复算方程，不要盲跑 Sage。
- Sage/LLL/Z3 长时间无解：检查变量界、泄露位宽、模数和方程是否对应真实代码；格问题转 [lattice-and-lwe.md](lattice-and-lwe.md)，PRNG 约束转 [prng-z3-lcg-and-timing-attacks.md](prng-z3-lcg-and-timing-attacks.md)。
- RsaCtfTool 入口不可用或没有覆盖：改用 FactorDB、SageMath 或题目专用脚本，并判断是否是 specialized prime/oracle/padding/fault；转 [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)。
- 证据来自 Web token、JSON parser、签名接口或 PCAP：保留 crypto 方程，但把请求/协议层转到 Web、Forensics 或 Pentest 对应 family。

## 详细清单

### FactorDB Web/API 与可选 MCP

默认使用 FactorDB Web/API。每次使用前检查当前会话的 callable 工具清单；只有实际出现相应 FactorDB MCP 方法时，才按下表调用，不要从本文假设连接已存在。

| 工具 | 功能 | 调用方式 |
|---|---|---|
| `factordb_query` | 查询大整数分解状态和已知因子 | 仅当 MCP callable；传入 `number` 参数 |
| `factordb_query_by_id` | 按数据库 ID 查询分解结果 | 仅当 MCP callable；传入 `id` 参数 |
| `factordb_report_factor` | 向 FactorDB 提交新发现的因子 | 仅当 MCP callable，且提交外部数据前确认；传入 `number` + `factor` |
| `factordb_report_factor_by_id` | 按 ID 向 FactorDB 提交因子 | 仅当 MCP callable，且提交外部数据前确认；传入 `id` + `factor` |

### Python 包（ctf-tools conda）

从 `pwsh` 直接调用：

```pwsh
wsl /home/kali/miniforge3/bin/conda run --no-capture-output -n ctf-tools python /path/to/script.py --example-argument value
```

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

从 `pwsh` 直接调用：

```pwsh
wsl /home/kali/miniforge3/bin/conda run --no-capture-output -n sage sage /path/to/script.sage
```

| 工具 | 版本 | 关键功能 | 典型用法 |
|---|---|---|---|
| **SageMath** | 10.9 | 全面符号数学环境 | `sage script.sage` |
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

### RsaCtfTool（Sage-Python 基础的独立 venv）

| 工具 | 路径与版本 | 功能 | 当前状态 |
|---|---|---|---|
| **RsaCtfTool** | 0.1.0；`/home/kali/RsaCtfTool/venv/bin/RsaCtfTool`；本地分支 `sage-compat-and-fixes`（Sage 兼容修复 + 四轮源码审计修复，已推 fork `KevinLee1231`） | RSA 自动攻击套件（Wiener/Hastad/Fermat/Pollard、格与 Sage 辅助攻击等） | 可用；venv Python 3.12.13，`sys.base_prefix=/home/kali/miniforge3/envs/sage`，`include-system-site-packages=false`；必须由外层 `conda run -n sage` 提供 `sage` 命令 |

自动化固定调用 venv 的绝对入口。RsaCtfTool 主进程使用 venv Python；声明 `required_binaries = ["sage"]` 的攻击通过 `PATH` 找到 `/home/kali/miniforge3/envs/sage/bin/sage`，Sage 启动器再使用 Sage 环境自己的 Python。不要把 RsaCtfTool 依赖安装进 Sage 基础环境：

```pwsh
wsl /home/kali/miniforge3/bin/conda run --no-capture-output -n sage /home/kali/RsaCtfTool/venv/bin/RsaCtfTool --publickey /path/to/key.pub --attack wiener --private
```

Sage 子进程攻击统一设有 180s 超时下限（`max(--timeout, 180)`；qicheng 保持 900s 下限匹配其 200 次 ECM 尝试预算），传入更大的 `--timeout` 时以传入值为准；对 1024 位以上密钥的格/ECM/QS 攻击应显式调大 `--timeout`。

确实需要同一 shell 内的 venv PATH 简写时，先由 `conda run` 建立 Sage PATH，再激活项目 venv；激活后 `python` 指向 venv，`sage` 仍指向 Sage Conda 环境：

```pwsh
wsl /home/kali/miniforge3/bin/conda run --no-capture-output -n sage /usr/bin/bash -c 'source /home/kali/RsaCtfTool/venv/bin/activate && RsaCtfTool --publickey /path/to/key.pub --attack wiener --private'
```

环境重建属于写操作，须先取得授权。重建时由 Sage 环境 Python 创建 venv，再始终使用 venv 的绝对 Python 安装；不要使用 `--system-site-packages`、裸 `pip` 或 `conda activate`：

```pwsh
wsl /home/kali/miniforge3/bin/conda run --no-capture-output -n sage python -m venv /home/kali/RsaCtfTool/venv
wsl /home/kali/miniforge3/bin/conda run --no-capture-output -n sage /home/kali/RsaCtfTool/venv/bin/python -m pip install -e /home/kali/RsaCtfTool
```

安装或 Sage 环境升级后按以下顺序检查。期望 `sys.executable` 位于项目 venv、`sys.base_prefix` 位于 Sage 环境、`shutil.which("sage")` 返回 Sage 绝对路径：

```pwsh
wsl /home/kali/miniforge3/bin/conda run --no-capture-output -n sage /home/kali/RsaCtfTool/venv/bin/python -c 'import shutil,sys; print(sys.executable); print(sys.base_prefix); print(shutil.which("sage"))'
wsl /home/kali/miniforge3/bin/conda run --no-capture-output -n sage /home/kali/RsaCtfTool/venv/bin/python -m pip check
wsl /home/kali/miniforge3/bin/conda run --no-capture-output -n sage /home/kali/RsaCtfTool/venv/bin/RsaCtfTool --help
```

若只激活 venv 后 `sage` 不存在，说明外层缺少 `conda run -n sage`；若 `sys.base_prefix` 不再指向 Sage 环境，或 Sage 环境更换了 Python 小版本，应删除并重建项目 venv，而不是手工改解释器软链接。

Sage 攻击脚本可用性（全部 10 个 `.sage` 已做 preparse+compile 静态检查并逐脚本实跑）：

| 脚本 | 状态 | 依据 |
|---|---|---|
| qs / ecm / ecm2 / smallfraction / small_crt_exp / boneh_durfee | 可用 | 构造弱密钥端到端实跑，因子/d 恢复且校验通过 |
| binary_polynomial_factoring | 可用 | 真实 Sage 子进程 pytest 通过 |
| roca_attack.py | 可执行 | 非 ROCA 输入正确输出 FAIL；完整分解路径需真 ROCA 密钥未测 |
| partial_d | 可用 | Coppersmith X 上界多档扫描覆盖因子位长超过 n/2 的场景；泄露 200 位弱密钥与作者样例回归全部命中 |
| lattice | 可用 | Coppersmith small_roots 已知高位分解；beta 取保守下界 `(pbits-1)/nbits` 保证 `factor >= n^beta` 前提成立，搜索窗多档扫描。平衡因子 96 位未知 8/8 命中 |
| qicheng | 可用 | 自包含标准 ECM（随机曲线 + 阶乘阶梯 + 模逆失败 gcd 提取）；timeout 强制下限 900s 匹配 attempts=200。实测命中 36 位因子；40 位以上建议优先用 GMP-ECM 的 ecm 攻击 |

neca、wolframalpha 攻击因对应二进制缺失在框架层 warning 后跳过。

### 算法库质量状态

- 项目全量测试套件 239 项全部通过、0 跳过（须由外层 `conda run -n sage` 提供 PATH；否则 sage 集成测试被 skip）。
- 经四轮源码审计修复并回归：QS 由逐点试除重写为素数幂筛线 + Hensel 提升的对数筛网（2.7–3x 加速）；mlucas 改 MSB-first Lucas 链；inv_mod_pow_of_2 重写为牛顿迭代；pollard_rho/hart/lehman 加固并统一 None 失败契约；same_n_huge_e 补 gcd(e1,e2)>1 处理；noveltyprimes 候选素性过滤；conspicuous_check 报告完整性；sage 攻击脚本缺失由 required_scripts 预检拦截（ecm.sage/ecm2.sage 曾在本地丢失，已从上游恢复）。行为由 `tests/test_audit_regressions.py` 固化。
- 核心算法人工审阅确认正确：Dixon 关系收集 + GF(2) 高斯消元 + 零空间依赖遍历、QS 因子基构建与关系筛选（负 Q(x) 的 -1 parity 处理正确）、euler 两平方和方法、fermat 增量式、wiener 连分数后处理、solve_partial_q 三重校验。
- 已知限制（使用时留意，非缺陷）：`dixon`/`kraitchik`/`lehman`/`hart` 为教学级实现，大数性能有限，实际分解任务优先 QS/SIQS/ECM/factordb；`binary_polynomial_factoring.sage` 依赖 `str(factor(...))` 打印格式解析，Sage 大版本升级时需复测。

### 系统全局命令（WSL Kali）

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| **hashcat** | `/usr/bin/hashcat` 7.1.2 | GPU 哈希破解（全类型支持） | `hashcat -m 0 -a 3 hash.txt ?l?l?l?l` |
| **john** | `/usr/sbin/john` 1.9.0-jumbo | CPU 哈希破解 / JWT secret 暴力 | `john hash.txt --wordlist=rockyou.txt` |
| **xxd** | `/usr/bin/xxd` | 十六进制转储与还原 | `xxd -p ciphertext \| head` |
| **openssl** | `/usr/bin/openssl` 3.6.3 | 加解密/签名/证书操作 | `openssl pkeyutl -decrypt -inkey key.pem -in ciphertext.bin` |
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
| **CyberChef** | `D:/CTF工具/CyberChef/CyberChef_v10.23.0.html` 或在线 https://gchq.github.io/CyberChef/ | 万能编解码工具（含 Magic 自动识别） | 浏览器打开 HTML，拖入文件或粘贴文本 |
| **fastcoll** | `D:/CTF工具/fastcoll/fastcoll_v1.0.0.5_exe/fastcoll_v1.0.0.5.exe` | MD5 相同前缀碰撞生成 | `fastcoll_v1.0.0.5.exe -p prefix -o out1 out2` |

### 在线工具

| 工具 | 地址 | 功能 | 适用边界 |
|---|---|---|---|
| **quipqiup** | https://quipqiup.com/ | 自动求解保留词边界的简单替换密码 | 已确认是古典 substitution cryptogram 时使用；不替代通用密码分析 |
