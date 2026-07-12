---
type: tooling
tags: [cross-category, tooling, triage, tools, environment]
skills: [ctf-solve-challenge]
updated: 2026-07-11
---

# Cross-Category Triage Tooling

本页为 `ctf-solve-challenge` 提供跨方向首检工具、调用层和失败转向。它不是第十五个方向，也不承担专项求解；证据足以判断决定性主障碍后，应立即进入 14 个正式 `ctf-*` 专项之一。

## 工具选择边界

### 入口选择

- 跨方向状态只用于证据不足或多个信号冲突，不是执行型后备方向。
- 首轮用轻量脚本、格式识别、约束检查和文件 triage；出现明确 Stego、Hardware、Cloud、Mobile 或 Blockchain 信号后进入对应专项。
- Jail 题优先枚举允许语法和可用 builtin，不要先上爆破。

### 不应停留在跨方向工具链的情况

- 已经有 HTTP、binary、crypto equation、PCAP/磁盘镜像或漏洞利用主线时，先转对应专项。
- “看起来杂”但需要真实服务攻击、反编译或密码学推导时，按决定性主障碍进入专项。
- 只有一次性剧情、flag 猜测或手工观察，不沉淀成跨方向工具入口。

### 补工具经验的触发条件

- raw 给出 WebSocket/WASM/canvas/存档格式等游戏状态机，并且工具链可复用。
- OCR、字体映射、组合字符或同形异义成为主要证据，而非一次性观察。
- SAT/ILP/SMT、递推或博弈搜索需要固定输入建模和失败判断。

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

- 当前没有明显缺口。跨方向工具页更需要守住“只做首检与路由”的边界，而不是继续扩容。

## 失败信号与转向

- 文件首检出现明确专项信号：立即转对应 skill，不要把跨方向状态当默认桶。
- 编码/QR/条码工具无结果：保存原始字节和中间图像；完整表示层编码转 [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md)，碎片/隐藏载荷转 Stego 页面，异常格式先看 [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md)。
- Jail 或 sandbox 只能得到部分输出：先记录可用语法、builtin、fd、错误回显和过滤阶段，再转 [pyjails.md](pyjails.md)、[bashjails.md](bashjails.md) 或 [interactive-containers-jails-and-solvers.md](interactive-containers-jails-and-solvers.md)。
- Z3/SAT/搜索脚本无解：检查变量域、位宽、目标函数和反馈 oracle；若状态递推或验证码反馈更强，转 [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md)。

## 详细清单

### ctf-tools conda 环境 (Python 包)

| 包名 | 版本 | 功能 | 典型用法 | 对应 skill |
|---|---|---|---|---|
| z3-solver | 4.13.0.0 | SMT 约束求解器 | `from z3 import *; Solver().add(constraints)` | 按约束来源进入 ctf-crypto / ctf-reverse / ctf-pwn / ctf-web |
| angr | 9.2.209 | 二进制符号执行框架 | `angr.Project('binary').explore(find=...)` | ctf-reverse / ctf-pwn |
| pwntools | 4.15.0 | CTF 漏洞利用工具库 | `from pwn import *` | ctf-pwn |
| sympy | 1.14.0 | 符号数学计算 | `sympy.solve(equation, x)` | ctf-crypto / ctf-reverse |
| gmpy2 | 2.3.0 | 高精度多精度算术 | `gmpy2.invert(a, n)` | ctf-crypto |
| numpy | 2.4.4 | 多维数组数值计算 | `np.fromfile('data', dtype=np.complex64)` | ctf-ai-ml / ctf-forensics / ctf-stego / ctf-hardware-embedded |
| scipy | 1.17.1 | 科学计算/信号处理 | `scipy.signal.spectrogram(audio)` | ctf-ai-ml / ctf-stego / ctf-hardware-embedded |
| Pillow | 11.3.0 | Python 图像处理 | `Image.open('puzzle.png')` | ctf-ai-ml / ctf-forensics / ctf-stego |
| matplotlib | 3.10.8 | 数据可视化 | `plt.scatter(x, y)` | 多专项共享；由数据来源决定路由 |
| requests | 2.33.1 | HTTP 客户端 | `requests.get(url)` | ctf-web / ctf-osint / ctf-cloud-infra |
| dnspython | 2.8.0 | DNS 协议 Python 实现 | `dns.resolver.resolve('example.com')` | ctf-web / ctf-forensics / ctf-osint / ctf-pentest |
| dnslib | 0.9.26 | DNS 报文构造与解析 | `dnslib.DNSRecord.parse(packet)` | ctf-web / ctf-forensics / ctf-malware |
| pycryptodome | 3.17 | 加密算法库 | `Crypto.Cipher.AES.new(key, ...)` | ctf-crypto |
| pyinstxtractor-ng | 2026.4.7 | PyInstaller 打包 exe 解包 | `pyinstxtractor-ng packed.exe` | ctf-reverse / ctf-malware |
| uncompyle6 | 3.9.3 | Python 2.7-3.8 bytecode 反编译 | `uncompyle6 compiled.pyc` | ctf-reverse |
| flask-unsign | 1.2.1 | Flask session 签名/解码 | `flask-unsign -d -c 'cookie'` | ctf-web |
| python-sat | 1.9.dev2 | SAT / MaxSAT / MUS 求解，适合约束与组合谜题 | `from pysat.solvers import Solver` | 工具不是方向；按模型来源进入 ctf-crypto / ctf-reverse / ctf-web |
| pywhat | 5.1.0 | 快速识别未知字符串、编码、哈希和格式 | `pywhat 'input'` | ctf-solve-challenge / ctf-crypto / ctf-forensics |
| pyzbar | 0.1.9 | Python 条码/二维码解码绑定 | `from pyzbar.pyzbar import decode` | ctf-stego / ctf-crypto |
| zxing-cpp | 3.0.0 | 多格式条码/二维码识别库 | `import zxingcpp` | ctf-stego / ctf-crypto |

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
