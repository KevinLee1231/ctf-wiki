---
type: family
tags: [forensics, family, map]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/cross-domain-forensics-technique-map.md
  - ../raw/forensics/SUCTF2026-forensicsWP.md
  - ../raw/forensics/SUCTF2026-LightNovelWP.md
updated: 2026-07-27
---

# Cross-Domain Technique Map

## 作用边界

题目跨越多个取证介质，单一页面不足以覆盖：例如 PCAP 里传磁盘镜像、内存 dump 中藏浏览器历史、视频里出现硬件信号、容器层里残留凭据。这个页面应作为“下一跳地图”，帮助快速选择更具体技巧，而不是承载完整解法。

## 识别信号

- 附件类型多，或者一个文件明显包含另一个领域的证据。
- 首轮 `file` / `binwalk` / `strings` 只说明“有嵌套”，但还不能确定最终路径。
- 多个候选方向都能解释一部分现象：网络、磁盘、内存、图像、音频、硬件信号交错出现。
- raw 中的技巧名更像索引：Docker、RAID、GIMP raw dump、I2C、USB HID、DNS trailing byte、TLS master key。

## 最小证据

- 至少完成载体拆分：知道当前是在 PCAP、镜像、内存、媒体、日志还是容器层。
- 对每一层保存一个可复现中间产物，例如导出的 TCP stream、mount 后目录树、frame dump、音频频谱图。
- 每次 pivot 都要能说明“为什么从 A 跳到 B”，例如协议里出现文件头、内存里出现 AES key、视频里出现 LED Morse。

## 分流流程

1. 先做低成本分层：`file`、`binwalk`、`7z l`、`tshark -r`、`strings`、`exiftool`。
2. 为每一层建立 evidence log：输入文件、导出命令、输出路径、观察到的 flag-like/secret-like 线索。
3. 选最强证据进入专项页：PCAP 走协议重组，磁盘走 Sleuth Kit，内存走 Volatility/raw carving，媒体走图像/音频/视频隐写。
4. 如果工具失败，改用原始字节/格式结构验证，不要被单个工具结论卡住。
5. 解出中间 secret 后再回到上一层验证它是否是密码、key、seed 或 flag。

## 跨域路线分流

| 跨域证据 | 下一跳判断 |
|---|---|
| 容器/镜像转文件系统 | Docker layer、SquashFS、disk image 先拆层，再看删除文件和历史命令。 |
| 网络转文件 | 从 PCAP 导出 HTTP/SMB/TCP stream，按 magic bytes 重组。 |
| 内存转密钥 | Volatility 失败时，直接在 dump 中找 key、session id、framebuffer 或脚本源码。 |
| 媒体转信号 | 视频/音频可能是 LED Morse、UART、SSTV、频率音符或打印轨迹。 |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| 扩展名、magic、容器边界或内嵌对象决定首检 | [file-format-and-embedded-payload-identification.md](file-format-and-embedded-payload-identification.md) |
| PCAP/协议分段、重传和 framing 需要会话重组 | [protocol-stream-reassembly-and-credential-extraction.md](protocol-stream-reassembly-and-credential-extraction.md) |
| 内存、VM snapshot 或 container layer 保留运行时/历史证据 | [memory-process-and-container-layer-recovery.md](memory-process-and-container-layer-recovery.md) |

## 常见陷阱

- 把跨域索引页当最终解法页，导致只浏览不验证。
- 每次换方向都丢失中间产物，无法复现。
- 工具扫描没有结果就结束，而不是回到格式结构和字节证据。
- 忽略“解出来的字符串可能只是下一层密码”。

## 关联技巧

- [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md)
- [3d-printing.md](3d-printing.md)
- [audio-frequency-and-archive-stego.md](audio-frequency-and-archive-stego.md)
- [blockchain-and-transaction-forensics.md](blockchain-and-transaction-forensics.md)
- [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)
- [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md)
- [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md)
- [pdf-png-gif-and-text-stego.md](pdf-png-gif-and-text-stego.md)
- [video-document-and-media-stego.md](video-document-and-media-stego.md)
- [keyboard-mouse-audio-and-physical-puzzles.md](keyboard-mouse-audio-and-physical-puzzles.md)
- [signals-and-hardware.md](signals-and-hardware.md)
- [linux-git-browser-and-container-forensics.md](linux-git-browser-and-container-forensics.md)
- [windows-registry-logs-and-credentials.md](windows-registry-logs-and-credentials.md)
- [structured-document-history-and-hidden-object-recovery.md](structured-document-history-and-hidden-object-recovery.md)

## WP 案例沉淀

本节只抽取 raw WP 中可复用的识别信号和下一跳，不替代原始题解正文。

| Raw WP | 复用信号 | 下一跳 |
|---|---|---|
| [SUCTF2026-forensicsWP](../raw/forensics/SUCTF2026-forensicsWP.md) | AD1 Windows 系统盘综合取证，证据集中在事件日志、Notepad TabState、应用缓存、聊天数据库和 Ollama 痕迹。 | [windows-registry-logs-and-credentials.md](windows-registry-logs-and-credentials.md)、[disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md) |
| [SUCTF2026-LightNovelWP](../raw/forensics/SUCTF2026-LightNovelWP.md) | PCAP 中出现 Kerberos/NTLM、MSRPC 与 TSCH 远程任务调度，先做 stream、凭据和会话密钥恢复，再解析计划任务与 PowerShell/AES 逻辑。 | [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md)、[windows-registry-logs-and-credentials.md](windows-registry-logs-and-credentials.md) |
| [0xGame2025-week2-ezShiro-wp](../raw/forensics/0xGame2025-week2-ezShiro-wp.md) | `rememberMe` 长 Base64 Cookie、已知默认密钥和 Java 反序列化 gadget 是 CVE-2016-4437 的关键识别链；流量分析时要区分漏洞入口与利用后的通信协议；本题的 Basic Base64 字段负责传命令，DNS 查询负责带出无回显结果。 | 本页对应路线 |
| [SekaiCTF2026-deadgame2-wp](../raw/forensics/SekaiCTF2026-deadgame2-wp.md) | 回放取证要先保证协议版本相容，再分别处理离散聊天事件和单位坐标证据。只读聊天会缺少中段；只看散点图又不知道 flag 的前后缀与拼接顺序。这里 MD5 不是替代分析的答案 oracle，而是用来裁决末尾重复分隔符的歧义；最终字符串必须同时满足事件时间线、可视化字形、特殊 flag 大小写和给定哈希。 | 本页对应路线 |
| [UMDCTF2017-image-crash-wp](../raw/forensics/UMDCTF2017-image-crash-wp.md) | PNG 能解压并不代表扫描线参数正确。若图像呈现从某行开始持续传播的错位，应检查每行第一个滤波器字节，并利用上下行连续性判断错误类型。本题只修改滤波器元数据，没有猜测或改写像素正文；最终图像和官方摘要共同验证了修复。 | 本页对应路线 |
| [UMDCTF2017-leetfiltration-wp](../raw/forensics/UMDCTF2017-leetfiltration-wp.md) | 网络隐蔽信道的逻辑符号不一定与数据包一一对应。TCP 是字节流协议，分段和合并都可能改变包边界；可靠做法是先按方向重组负载，再根据自描述的前缀规则解析 `0x13` 与 `0x1337`，最后按 8 bit 大端恢复字节。 | 本页对应路线 |
| [UMDCTF2018-producer-consumer-wp](../raw/forensics/UMDCTF2018-producer-consumer-wp.md) | 永久 WMI 事件订阅由“过滤器、消费者、绑定”三部分构成，只找到其中一个对象不足以说明完整持久化链。离线仓库分析时应先恢复对象关系，再解码消费者命令；外部工具只是解析载体，关键对象名、触发条件和最终命令都应记录进 WP。 | 本页对应路线 |
| [UMDCTF2019-matryoshka-wp](../raw/forensics/UMDCTF2019-matryoshka-wp.md) | 套娃题的核心不是无条件递归解压，而是每一层都重新做格式识别、结构检查和大小评估。遇到重复文件名要按归档条目读取；遇到极高压缩比要先列清单并选择性提取。路径名、扩展名和首个看似合理的对象都可能是诱饵，最终结论应由格式结构和官方摘要共同验证。 | 本页对应路线 |
| [UMDCTF2021-philip-2-wp](../raw/forensics/UMDCTF2021-philip-2-wp.md) | 邮件取证不能只看索引数据库。数据库适合搜索主题、正文、附件名和口令，真正附件还可能位于 mbox、IMAP 缓存或 MIME 数据中。本题应先通过 PID 与打开文件确定 profile，再恢复数据库和消息体，最后按 MIME 编码重组附件。 | 本页对应路线 |
| [UMDCTF2023-yara-trainer-gym-wp](../raw/forensics/UMDCTF2023-yara-trainer-gym-wp.md) | 把八条 YARA 条件拆成文件魔数、字符串、ELF 结构、熵和大小五类，再构造一个同时满足全部约束的合法 ELF；上传结果逐项显示规则命中状态时，可以把它当作约束 oracle，逐次只改变一个文件属性并观察反馈。 | 本页对应路线 |
| [UMDCTF2024-image-abomination-wp](../raw/forensics/UMDCTF2024-image-abomination-wp.md) | 针对 JPEG 熵编码失步，先鲁棒解码并做块级颜色/位置校正，再利用保留下来的缩略图引导像素级恢复；JPEG 头和尺寸正常、图像只在前部可见、后续大面积灰块且解码器提示数据段提前结束时，应怀疑扫描码流比特错误。 | 本页对应路线 |
| [WMCTF2022-hacked-by-l1near-wp](../raw/forensics/WMCTF2022-hacked-by-l1near-wp.md) | 本题核心是按协议层级拆 WebSocket 压缩帧：先解析 FIN/RSV/opcode 和 payload length，再按 mask key 还原客户端 payload，最后用 raw DEFLATE 解压 `permessage-deflate` 数据。 | 本页对应路线 |
| [MoeCTF2024-特工luo-闻风而动-wp](../raw/reverse/MoeCTF2024-特工luo-闻风而动-wp.md) | 这题不是单独的 Wi-Fi 跑字典：keygen 逆向负责把候选空间压缩到可行规模，握手包只负责离线验证，最终还要按客户端数据流逆序解开 RC4 和 DES。文件名既是载荷名也是外层口令，是最容易漏掉的关联。 | 本页对应路线 |

## 原始资料
- [cross-domain-forensics-technique-map.md](../raw/forensics/cross-domain-forensics-technique-map.md)
- [SUCTF2026-forensicsWP.md](../raw/forensics/SUCTF2026-forensicsWP.md)
- [SUCTF2026-LightNovelWP.md](../raw/forensics/SUCTF2026-LightNovelWP.md)
