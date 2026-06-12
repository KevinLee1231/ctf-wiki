---
type: tooling
tags: [misc, tooling, tools, environment]
skills: [ctf-misc]
updated: 2026-05-21
---

# Misc Tooling

本页记录 `ctf-misc` 方向的本机工具清单、调用层、路径和适用边界。`SKILL.md` 只保留首轮工具摘要；需要详细路径、环境和专项工具说明时读取本页。

## 调用层与覆盖状态

### 非交互调用原则

- `ctf-misc` 是后备方向，不应在主障碍明显属于 Web/Pwn/Reverse/Crypto/Forensics 时抢先使用。
- 首轮用轻量脚本、编码工具、约束求解和文件 triage；需要专业 RF/音频/图像能力时再查对应技巧页。
- Jail 题优先枚举允许语法和可用 builtin，不要先上爆破。

### 知识页覆盖状态

- 当前覆盖 pyjail、bash jail、RF/SDR、DNS、编码、esolang、交互容器、物理谜题和轻量 solver。
- 缺口主要是游戏/图形类谜题、OCR/字体谜题、浏览器小游戏状态机与更多交互协议。

### 后续补强方向

- Game/state puzzle：WebSocket、WASM、canvas、存档格式。
- OCR/font/unicode：字体映射、组合字符、同形异义。
- Solver patterns：SAT/ILP/SMT、递推、博弈状态搜索。

## 本机工具清单（按使用时机）

### 首轮常用

| 工具 | 为什么放在首轮 |
|---|---|
| `file` / `strings` / `xxd` | 先判断它是不是普通编码、容器或伪装格式 |
| `zbarimg` | QR / 条形码题最直接 |
| `ctf-tools` Python | 编码、约束、小脚本谜题最常用的统一脚本环境 |

### 专项按需

- 约束/符号：`z3-solver`、`sympy`、`python-sat`
- 二进制/VM 辅助：`angr`、`pwntools`
- 信号/图像：`numpy`、`scipy`、`Pillow`、`matplotlib`
- 协议与网络：`requests`、`dnspython`、`dnslib`
- 打包/字节码：`pyinstxtractor-ng`、`uncompyle6`
- 识别/条码/esolang：`pywhat`、`pyzbar`、`zxing-cpp`、`npiet`、`npietedit`
- 交叉题型辅助：`pycryptodome`、`flask-unsign`

### 当前未装 / 建议按需补装

- 当前没有明显缺口。Misc 更需要守住“不是默认入口”的边界，而不是继续扩容。

## 详细清单

### ctf-tools conda 环境 (Python 包)

| 包名 | 版本 | 功能 | 典型用法 | 对应 skill |
|---|---|---|---|---|
| z3-solver | 4.13.0.0 | SMT 约束求解器 | `from z3 import *; Solver().add(constraints)` | ctf-misc / ctf-crypto |
| angr | 9.2.209 | 二进制符号执行框架 | `angr.Project('binary').explore(find=...)` | ctf-misc / ctf-reverse |
| pwntools | 4.15.0 | CTF 漏洞利用工具库 | `from pwn import *` | ctf-pwn / ctf-misc |
| sympy | 1.14.0 | 符号数学计算 | `sympy.solve(equation, x)` | ctf-misc / ctf-crypto |
| gmpy2 | 2.3.0 | 高精度多精度算术 | `gmpy2.invert(a, n)` | ctf-crypto / ctf-misc |
| numpy | 2.4.4 | 多维数组数值计算 | `np.fromfile('data', dtype=np.complex64)` | ctf-misc / ctf-forensics |
| scipy | 1.17.1 | 科学计算/信号处理 | `scipy.signal.spectrogram(audio)` | ctf-misc |
| Pillow | 11.3.0 | Python 图像处理 | `Image.open('puzzle.png')` | ctf-misc / ctf-forensics |
| matplotlib | 3.10.8 | 数据可视化 | `plt.scatter(x, y)` | ctf-misc |
| requests | 2.33.1 | HTTP 客户端 | `requests.get(url)` | ctf-misc / ctf-web |
| dnspython | 2.8.0 | DNS 协议 Python 实现 | `dns.resolver.resolve('example.com')` | ctf-misc |
| dnslib | 0.9.26 | DNS 报文构造与解析 | `dnslib.DNSRecord.parse(packet)` | ctf-misc |
| pycryptodome | 3.17 | 加密算法库 | `Crypto.Cipher.AES.new(key, ...)` | ctf-crypto / ctf-misc |
| pyinstxtractor-ng | 2026.4.7 | PyInstaller 打包 exe 解包 | `pyinstxtractor-ng packed.exe` | ctf-misc / ctf-reverse |
| uncompyle6 | 3.9.3 | Python 2.7-3.8 bytecode 反编译 | `uncompyle6 compiled.pyc` | ctf-misc |
| flask-unsign | 1.2.1 | Flask session 签名/解码 | `flask-unsign -d -c 'cookie'` | ctf-misc / ctf-web |
| python-sat | 1.9.dev2 | SAT / MaxSAT / MUS 求解，适合约束与组合谜题 | `from pysat.solvers import Solver` | ctf-misc |
| pywhat | 5.1.0 | 快速识别未知字符串、编码、哈希和格式 | `pywhat 'input'` | ctf-misc |
| pyzbar | 0.1.9 | Python 条码/二维码解码绑定 | `from pyzbar.pyzbar import decode` | ctf-misc / ctf-forensics |
| zxing-cpp | 3.0.0 | 多格式条码/二维码识别库 | `import zxingcpp` | ctf-misc / ctf-forensics |

### 系统全局命令（WSL Kali）

| 工具 | 路径 | 版本 | 功能 | 典型用法 |
|---|---|---|---|---|
| zbarimg | /usr/bin/zbarimg | 0.23.93 | 二维码/条形码解码 | `zbarimg qrcode.png` |
| qrencode | /usr/bin/qrencode | 4.1.1 | 二维码生成 | `qrencode -o out.png "flag{...}"` |
| ffmpeg | /usr/bin/ffmpeg | 8.1 | 音视频转换/频谱图 | `ffmpeg -i audio.wav -lavfi showspectrumpic spec.png` |
| sox | /usr/bin/sox | — | 音频处理 / 频谱 / 反转 | `sox audio.wav -n spectrogram` |
| qsstv | /usr/bin/qsstv | — | SSTV 慢扫描电视解码 | GUI 打开并载入音频 |
| exiftool | /usr/bin/exiftool | 13.50 | 文件元数据读写 | `exiftool audio.wav` |
| zsteg | /usr/local/bin/zsteg | 0.2.14 | PNG/BMP 位面隐写检测 | `zsteg -a image.png` |
| binwalk | /usr/bin/binwalk | — | 文件嵌入提取与分析 | `binwalk -Me file.bin` |
| nmap | /usr/bin/nmap | 7.99 | 端口扫描/服务检测 | `nmap -sV target` |
| nc | /usr/bin/nc | — | TCP/UDP 网络连接 | `nc target 1337` |
| 7z | /usr/bin/7z | 26.00 | 通用压缩/解压 | `7z x archive.7z` |
| netpbm | (系统包) | 11.13.03 | 图像格式转换工具集 | `pamscale` / `pnmtopng` / `ppmtopgm` |
| npiet | /usr/local/bin/npiet | — | Piet esolang 解释器 | `npiet program.png` |
| npietedit | /usr/local/bin/npietedit | — | Piet 程序图像编辑辅助工具 | `npietedit program.png` |
| wasm2wat | /usr/bin/wasm2wat | — | WebAssembly 二进制转文本 | `wasm2wat module.wasm -o module.wat` |
| wat2wasm | /usr/bin/wat2wasm | — | WebAssembly 文本转二进制 | `wat2wasm module.wat -o module.wasm` |

### 数学环境

| 工具 | 路径 | 版本 | 功能 | 典型用法 |
|---|---|---|---|---|
| SageMath | conda 环境 sage | 10.7 | 完整数学软件系统 | `sage script.sage` |

### 独立工具

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| pycdc | /home/kali/pycdc/build/pycdc | Python 3.9+ bytecode 反编译 | `pycdc file.pyc` |
| PySAT helper scripts | `/home/kali/.local/bin/{approxmc.py,bbscan.py,bica.py,fm.py,genhard.py,lbx.py,lsu.py,mcsls.py,models.py,musx.py,optux.py,primer.py,rc2.py,unigen.py}` | SAT / MUS / MaxSAT / 约束实例分析辅助脚本 | 约束类题目按需直接调用对应脚本 |

### 未安装但题目中可能需要的工具（参照表）

| 工具 | 用途 | 备用方案 |
|---|---|---|
| segno | 高级 QR 码生成（Micro QR、SVG） | 使用 qrencode 命令行 |
| hlextend | SHA-256/MD5 长度扩展攻击 | 手动实现或 conda 安装 |
| smspdu | SMS PDU 编码解码 | 手工解析 |
| zxing（Java CLI） | 特殊条码格式（MaxiCode 等） | 本机已装 `zxing-cpp`，需要 Java CLI 时再按需补装 |
| pytesseract | OCR 文字识别 | 在线 OCR 服务 |
