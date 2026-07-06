---
type: family
tags: [forensics, family, pcap, network, credential, protocol, tls, hid]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/pcap-protocol-credential-recovery.md
  - ../raw/forensics/SU_LightNovelWP.md
  - ../raw/misc/VNCTF2026-ez-iot-wp.md
updated: 2026-07-06
---

# PCAP 协议、凭据与文件恢复技巧族

## 作用边界

本页是 PCAP / network forensics family，负责从协议分布、会话方向、可导出对象、凭据材料和加密/编码层判断下一跳。

它不处理所有取证附件；只有核心证据来自网络流量、抓包、会话重组、传输对象、凭据或网络 covert channel 时才进入本页。导出出的文件、密钥或媒体应继续转到对应 forensics / crypto / web 页面。

## 识别信号

- 附件是 `pcap/pcapng`，或取证任务中包含网络流量、TLS keylog、USB HID、WiFi、SMB、DNS、HTTP、邮件或自定义 TCP/UDP。
- flag 可能不在明文响应中，而在文件传输、凭据登录、cookie/token、会话重组、加密流量解密或 covert channel 中。
- 存在多个协议层：压缩包分片、HTTP upload/download、DNS 子域编码、SMB 文件、TLS master key、USB 键盘鼠标事件。
- 需要从流量导出工件，再交给 forensics / crypto / web 继续处理。

## 最小证据

- 已保存协议统计、conversation 列表、top endpoints、时间范围和可疑 stream 编号。
- 至少导出一个可复现对象：HTTP body、TCP stream、DNS 查询序列、SMB 文件、TLS 解密会话或 USB HID 事件表。
- 明确目标是恢复文件、恢复凭据、还原命令、解密 TLS/WiFi，还是分析 covert channel。
- 若涉及解密，已定位 keylog、证书、密码、握手参数或可爆破材料。

## 分流流程

1. 先做协议分布和时间线，不直接搜索 flag 了事。
2. 根据协议选择导出方式：Wireshark objects、tshark fields、tcpflow、NetworkMiner、bulk_extractor、aircrack、USB HID decoder。
3. 对导出物保留哈希、文件名、stream 编号和命令，避免后续无法回溯。
4. 如果得到的是凭据/token/cookie，回到 Web 或 Pentest 路线验证用途。
5. 如果得到的是加密/压缩/损坏文件，再 pivot 到对应 forensics 或 crypto 页面。

## 路线分流

| 变体 | 优先证据 | 下一跳页面 | 失败后 pivot |
|---|---|---|---|
| HTTP 文件/凭据恢复 | `http`, `multipart`, `Set-Cookie`, `Authorization` | [network-covert-auth-and-reassembly.md](network-covert-auth-and-reassembly.md) | 如果对象导出失败，改用 TCP stream 和手工 carving。 |
| TLS 解密 | keylog、master secret、私钥、握手完整 | [filesystems-memory-dumps-and-raid.md](filesystems-memory-dumps-and-raid.md) | 没有 key 时查内存 dump、浏览器 profile 或凭据泄漏。 |
| DNS/ICMP covert channel | 高频子域、长 label、固定长度 payload | [dns.md](dns.md) | 如果编码不明显，按时间间隔、base32/base64/hex 和排序恢复。 |
| USB HID / 外设事件 | USB interrupt transfer、键盘鼠标 usage ID | [peripheral-capture.md](peripheral-capture.md) | 若轨迹不成字，转坐标可视化或 chord/stenography。 |
| WiFi/WPA/WEP | 802.11 握手、SSID、EAPOL、弱 WEP | [rf-sdr.md](rf-sdr.md) | 缺握手时查 deauth、PMKID 或是否已有密码线索。 |
| Kerberos/NTLM + MSRPC/TSCH | NetNTLMv2、Kerberos ticket、DCE/RPC opnum、TaskSchedulerService stream | [windows-registry-logs-and-credentials.md](windows-registry-logs-and-credentials.md) | 先恢复凭据或会话密钥；若 RPC stub 加密，保留 frame、stream 和 key 派生脚本。 |
| 分片/损坏 pcap | magic 损坏、截断、乱序、多段文件 | [file-signatures-and-flag-artifact-hunting.md](file-signatures-and-flag-artifact-hunting.md) | 先修 pcap，再重组；不要直接对损坏流量做上层解析。 |
| ESP-NOW/vendor action frames | 固件里能恢复 magic、seq、total、IV 和加密模式 | [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md) | 先用固件确定应用层边界，再从 raw capture 切帧、解密和按序重组。 |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [SU_LightNovelWP](../raw/forensics/SU_LightNovelWP.md) | AD/Kerberos/NTLM + TSCH 流量应先按 conversation 和 stream 定位认证握手，再用恢复出的凭据/会话密钥解 RPC stub，最后把计划任务 XML 与 PowerShell/AES 逻辑交给 Windows artifact 路线。 |
| [VNCTF2026-ez-iot-wp](../raw/misc/VNCTF2026-ez-iot-wp.md) | ESP-NOW 捕获不能直接把整帧当密文；先定位 vendor frame 和应用层 magic，再按 `seq`、`total`、IV、AES-CBC 密文切块重组 PNG。 |
| [D3CTF2021-easy-quantum-wp](../raw/misc/D3CTF2021-easy-quantum-wp.md) | PCAP 中传输 pickle/numpy 量子态和测量基，先按流量结构恢复 BB84 密钥再异或密文。 |
| [D3CTF2021-robust-wp](../raw/misc/D3CTF2021-robust-wp.md) | HTTP/3/QUIC 流量和媒体信号混合，先重组网络层数据，再转频谱或 payload 提取。 |

## 常见陷阱

- 只用 Wireshark GUI 手工点选，不保存 tshark 命令和 stream 编号。
- 只搜 `flag{`；真实 flag 可能在导出文件、解密后流量或登录后的二次请求里。
- 忽略时序和方向，导致把客户端上传误认为服务端下载。
- TLS 解不开就停止；keylog、内存、浏览器 profile、私钥和密码线索都可能在同题其它附件里。
- 对 DNS/USB HID 只按文本看；这类流量经常需要转时间序列、坐标或按 usage table 解码。


## 关联技巧

- [cross-domain-forensics-technique-map.md](cross-domain-forensics-technique-map.md)
- [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)
- [file-signatures-and-flag-artifact-hunting.md](file-signatures-and-flag-artifact-hunting.md)
- [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md)
- [network-covert-auth-and-reassembly.md](network-covert-auth-and-reassembly.md)
- [peripheral-capture.md](peripheral-capture.md)
- [keyboard-mouse-audio-and-physical-puzzles.md](keyboard-mouse-audio-and-physical-puzzles.md)
- [signals-and-hardware.md](signals-and-hardware.md)
- [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)
- [forensics-tooling.md](forensics-tooling.md)

## 原始资料

- [pcap-protocol-credential-recovery.md](../raw/forensics/pcap-protocol-credential-recovery.md)
- [SU_LightNovelWP.md](../raw/forensics/SU_LightNovelWP.md)
- [VNCTF2026-ez-iot-wp.md](../raw/misc/VNCTF2026-ez-iot-wp.md)
