---
type: tooling
tags: [crypto, tooling, tools, environment]
skills: [ctf-crypto]
---

# Crypto Tooling

本页维护 Crypto 工具的当前状态、路径、调用和失败处理。先按参数与 oracle 选择算法；工具已安装不构成进入大规模分解、格规约或爆破的理由。

## 平台与工具选择

普通编码、对称密码、模运算、符号数学和 SMT 使用 Windows venv `D:/文档/新建文件夹/venv/Scripts/python.exe`（Python 3.14.3）。只有已建立的问题确需当前 Sage 或 Linux 专项工具时才进入 WSL；`ctf-tools` 不再是通用密码包环境。

| Windows venv 包 | 版本 | 作用与脚本入口 |
|---|---|---|
| `pycryptodome` | `3.23.0` | AES/RSA/ChaCha20、padding、MAC；`from Crypto.Cipher import AES` |
| `cryptography` | `46.0.7` | 证书、密钥、签名与密码协议组件 |
| `gmpy2` | `2.3.0` | 大整数、GCD、模逆和整数根；`gmpy2.invert(e, phi)` |
| `sympy` | `1.14.0` | 符号方程、素数和小规模因式分解 |
| `z3-solver` | `4.16.0.0` | 位向量、整数和布尔约束；先给出变量域与已知测试 |
| `numpy` | `2.4.4` | 数值数组和统计；精确模运算不要误用浮点线性代数 |

```pwsh
& "D:/文档/新建文件夹/venv/Scripts/python.exe" "D:/题目路径/solve.py"
& "D:/文档/新建文件夹/venv/Scripts/python.exe" -c 'import gmpy2; from Crypto.Cipher import AES; print(gmpy2.invert(3,11)); print(AES.block_size)'
```

普通 XOR、Base64、hex 和序列化可用标准库与短脚本完成，不为这些操作转到 WSL。当前没有可直接调用的 Ciphey 或哈希长度扩展工具链；需要某个专项包时先评估 Windows 新版本或最小实现，不沿用已清理的包清单。

## FactorDB 与在线入口

RSA 模数或可疑合数出现时，在 [FactorDB](https://factordb.com/) 页面输入完整十进制整数，查询是否已有因子，并在本地验证因子乘积与素性。服务可达性在使用时核验；当前会话未暴露 FactorDB MCP，后续只有实际发现对应接口时才按其 schema 调用，不猜测工具名，也不把查询自动扩展为提交因子。

CyberChef 本地页面为 `D:/CTF工具/CyberChef/CyberChef_v10.23.0.html`（10.23.0），适合编码链、字节转换和小规模格式实验。古典单表替换且词边界有意义时可用 quipqiup；在线服务状态与输入是否适合提交需在实际使用时核对。

```pwsh
Start-Process "D:/CTF工具/CyberChef/CyberChef_v10.23.0.html"
```

## SageMath 与格工具

| 工具 | 当前版本与位置 | 使用边界 |
|---|---|---|
| SageMath | `10.9`；WSL Conda `sage` | 数论、有限域、ECC、Coppersmith、精确线性代数 |
| `fpylll` | `0.6.4`；同一 `sage` 环境 | LLL/BKZ、CVP/SVP；不在 `ctf-tools` |
| `fplll` CLI | `/usr/bin/fplll`；APT `fplll-tools 5.5.0-2` | 文本矩阵的独立格规约，不需要 Python 包 |

所有 WSL Conda 调用使用绝对入口，不激活环境。Windows 文件路径在 WSL 中改用 `/mnt/d/...`：

```pwsh
wsl -d kali-linux --exec /home/kali/miniforge3/bin/conda run --no-capture-output -n sage sage "/mnt/d/题目路径/solve.sage"
wsl -d kali-linux --exec /home/kali/miniforge3/bin/conda run --no-capture-output -n sage sage -c 'print(matrix(ZZ, [[1,1,1],[-1,0,2],[3,5,6]]).LLL())'
wsl -d kali-linux --exec /home/kali/miniforge3/bin/conda run --no-capture-output -n sage python -c 'from fpylll import IntegerMatrix, LLL; m=IntegerMatrix.from_matrix([[1,1,1],[-1,0,2],[3,5,6]]); print(LLL.reduction(m))'
wsl -d kali-linux --exec /usr/bin/fplll -a lll "/mnt/d/题目路径/matrix.txt"
```

`fplll` 的矩阵输入形如 `[[1 1 1] [-1 0 2] [3 5 6]]`。Sage 脚本中可按问题使用 `M.LLL()`、`M.BKZ(block_size=20)`、`f.small_roots(...)`、`EllipticCurve(GF(p), [a,b])`、`matrix(GF(2), data).solve_right(v)`。规约结果仍须代回原方程验证。

## RsaCtfTool 独立 venv

RsaCtfTool `0.1.0` 的入口为 `/home/kali/RsaCtfTool/venv/bin/RsaCtfTool`。项目 venv 基于 Sage Python 3.12，`include-system-site-packages=false`；外层 `conda run -n sage` 提供 Sage 子进程，Python 依赖仍在项目 venv。CLI 帮助可用，具体攻击应按题目输入单独验证。

```pwsh
wsl -d kali-linux --exec /home/kali/miniforge3/bin/conda run --no-capture-output -n sage /home/kali/RsaCtfTool/venv/bin/RsaCtfTool --help
wsl -d kali-linux --exec /home/kali/miniforge3/bin/conda run --no-capture-output -n sage /home/kali/RsaCtfTool/venv/bin/RsaCtfTool --publickey "/mnt/d/题目路径/key.pub" --attack wiener --private
wsl -d kali-linux --exec /home/kali/miniforge3/bin/conda run --no-capture-output -n sage /home/kali/RsaCtfTool/venv/bin/python -c 'import shutil,sys; print(sys.executable); print(sys.base_prefix); print(shutil.which("sage"))'
wsl -d kali-linux --exec /home/kali/miniforge3/bin/conda run --no-capture-output -n sage /home/kali/RsaCtfTool/venv/bin/python -m pip check
```

当前源码的 Sage 攻击使用 180s 最低超时，qicheng 使用 900s 下限；把 `--timeout` 设小不能保证立即退出。长时间计算先明确单项预算，不默认 `--attack all`。已知因子高位可考虑 lattice，私钥低位泄露可考虑 partial_d；教学级 dixon/kraitchik/lehman/hart 不适合替代大数分解引擎。

找不到 `sage` 时先核对外层调用；解释器或基础环境漂移时先读 venv 配置和依赖检查结果。环境重建属于 WSL 安装，必须说明目标、依赖与影响并取得明确同意；不能手改解释器链接或把依赖装进 Sage 基础环境。

## YAFU、HashClash 与系统工具

| 工具 | 位置与当前状态 | 作用 |
|---|---|---|
| YAFU | `/usr/local/bin/yafu`，手动部署；stdin 分解小样本可用，动态库均解析成功 | 自动整数分解；依赖 APT `libecm.so.1`、`libgmp.so.10` |
| HashClash | `/home/kali/hashclash/scripts/cpc.sh`、`/home/kali/hashclash/bin/md5_fastcoll`；现存本地编译入口 | Linux 选择前缀碰撞流程；耗时取决于前缀与参数 |
| Windows fastcoll | `D:/CTF工具/fastcoll/fastcoll_v1.0.0.5_exe/fastcoll_v1.0.0.5.exe` | MD5 相同前缀碰撞；能满足问题时优先本地 Windows |
| OpenSSL | `/usr/bin/openssl`；APT `3.6.3-1` | 密钥、证书、加解密与签名检查 |
| hashid | `/usr/bin/hashid`；APT `3.1.4-5` | 哈希格式候选识别，不能唯一判定算法 |
| hashcat | `/usr/bin/hashcat`；APT `7.1.2+ds1-4` | 哈希口令恢复；包存在不代表 GPU 后端可用 |
| John | `/usr/sbin/john`；APT `1.9.0-Jumbo-1+git20211102-0kali11` | CPU 口令恢复及格式转换 |

YAFU 非 TTY 调用使用 stdin，不把参数表达式误当成交互输入。它会在工作目录写日志，因此先切到本次题目目录；下面的小整数仅用于检查调用：

```pwsh
wsl -d kali-linux --exec bash -c 'cd "/mnt/d/题目路径" && printf "%s\n" "factor(8051)" | /usr/local/bin/yafu -threads 1'
& "D:/CTF工具/fastcoll/fastcoll_v1.0.0.5_exe/fastcoll_v1.0.0.5.exe" -p "D:/题目路径/prefix.bin" -o "D:/题目路径/a.bin" "D:/题目路径/b.bin"
wsl -d kali-linux --exec /usr/bin/openssl version
wsl -d kali-linux --exec /usr/bin/hashcat -I
```

HashClash 选择前缀任务获准运行后，在题目工作目录调用：

```pwsh
wsl -d kali-linux --exec bash -c 'cd "/mnt/d/题目路径" && /home/kali/hashclash/scripts/cpc.sh prefix1.bin prefix2.bin'
```

YAFU 的运行库仍有用途，不能因 Windows 有大整数 Python 包就删除；`libecm1` 不等于已经安装独立 `ecm` 命令。口令恢复使用题目自己的字典，不假定 rockyou 已存在。

## 失败处理与转向

- FactorDB、GCD、小指数和近似平方根无结果：回到 [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md) 重列已知量、未知量和可复算方程。
- LLL/Z3 无解或规模过大：核对变量界、位宽、模数及整数/浮点类型；格问题看 [lattice-and-lwe.md](lattice-and-lwe.md)，随机数看 [prng-z3-lcg-and-timing-attacks.md](prng-z3-lcg-and-timing-attacks.md)。
- RsaCtfTool 不覆盖或失败：保留具体报错，转题目专用脚本与 [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)。
- 缺通用 Python 包先评估 Windows 兼容版本；WSL 新增、升级或重装须先说明为何必须 Linux、安装方式、环境与影响，并取得明确同意。
