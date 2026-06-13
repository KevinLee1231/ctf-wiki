---
type: family
tags: [forensics, family, windows, registry, logs, credentials, timeline]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/windows-registry-logs-and-credentials.md
updated: 2026-06-12
---

# Windows Registry, Logs and Credentials

## 作用边界

本页是 Windows 末端痕迹 family，用于从注册表、事件日志、SAM/SYSTEM、NTFS 元数据、USN Journal、浏览器数据、Defender 日志、WMI 和内存凭据中恢复时间线、账号、密码、执行痕迹或隐藏文件。

它不替代磁盘/内存总取证页，也不替代 PCAP 凭据恢复页。当前证据已经明确落在 Windows 主机侧 artifacts 时，从本页做二级分流；若只是“有一个镜像或内存 dump”，先看 [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)。

## 共同识别信号

- 附件包含 `SAM`、`SYSTEM`、`SOFTWARE`、`NTUSER.DAT`、`.evtx`、`$MFT`、`$UsnJrnl:$J`、`OBJECTS.DATA`、`MPLog`、Prefetch、Chrome/Edge/Firefox profile 或 Windows 内存镜像。
- 题目提示用户创建、RDP、WMI、wmiexec、PowerShell、日志清理、浏览器凭据、回收站、ADS、Defender、恶意持久化或反取证。
- 目标不是单纯 carving 文件，而是重建“谁在什么时候做了什么”，或从系统状态里恢复 credential/secret。

## 最小证据

- 明确 artifact 来源：磁盘镜像、内存 dump、导出的 hive、单个日志文件还是浏览器 profile。
- 保留时间基准：Windows FILETIME、Unix time、USN timestamp、Chrome WebKit timestamp 或事件日志时间要统一到同一时区。
- 对凭据类证据，确认依赖关系：SAM 需要 SYSTEM boot key；Chrome Login Data 需要 Local State/DPAPI key；Firefox 密码需要 `key4.db` 和 `logins.json`。
- 对 anti-forensics，不能只说日志缺失；需要有替代证据，如 USN、MFT、SAM key timestamp、Defender MPLog、PowerShell history 或 Prefetch。

## 首轮路由

| 证据形态 | 先做什么 | 下一跳 |
|---|---|---|
| `.evtx`、RDP、用户创建/删除事件 | 按 Event ID 和 channel 建时间线，优先看 1102、4720、4724、4781、1149、21/24。 | [forensics-tooling.md](forensics-tooling.md) |
| `SAM` + `SYSTEM` 或内存中 hive | 提取 boot key 和 NTLM hash，再判断是破解、pass-the-hash 还是账号创建时间线。 | [network-covert-auth-and-reassembly.md](network-covert-auth-and-reassembly.md) |
| `$MFT`、USN、Recycle Bin、ADS | 先确认文件创建/删除/重命名/命名数据流，再决定恢复内容还是解释行为。 | [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md) |
| Chrome/Edge/Firefox profile | 区分 History、Cookies、Login Data、Local Storage、session restore；密码解密先找 master key。 | [linux-git-browser-and-container-forensics.md](linux-git-browser-and-container-forensics.md) |
| 日志被清理或攻击者擦痕迹 | 用 USN、MFT、SAM key timestamp、Defender MPLog、PowerShell history、Prefetch 重建旁路时间线。 | [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md) |
| Windows 内存 dump | 先用 Volatility 做 `pslist/psxview/cmdline/clipboard/hashdump/netscan`，再按进程或 hive 深挖。 | [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md) |
| WMI persistence 或 wmiexec 痕迹 | 检查 WMI repository、USN 中 `__<timestamp>.<random>` 输出文件、WMIPRVSE Prefetch 和 Defender 记录。 | [linux-git-browser-and-container-forensics.md](linux-git-browser-and-container-forensics.md) |

## 合并与拆分结论

- 保留为 `family`：raw 同时覆盖日志、注册表、SAM、NTFS、浏览器、Defender、WMI、内存凭据和反取证，核心价值是证据源分流。
- 不并入 [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)：后者负责镜像/内存/VM 容器的入口判断，本页负责 Windows artifact 的细分路线。
- 暂不拆成独立 Event Log、Registry、NTFS、Browser credential 页面：当前 raw 中它们经常共同支撑同一时间线，拆分会削弱查询路径；后续某类积累多篇 WP 后再拆 technique。

## 常见陷阱

- 时间戳格式混用，导致把 Chrome、USN、FILETIME 和 Unix 时间线排错。
- 只看 Security.evtx；RDP、Defender、PowerShell、WMI、Prefetch 和 USN 往往在日志清理后更有价值。
- 提取 Chrome/Edge 密码时只拿 `Login Data`，没有 Local State/DPAPI key。
- 把 ADS 当成普通文件缺失；NTFS 镜像里要用 `fls`/`istat`/`icat` 看命名数据流。
- 在内存题里只跑 `strings`，没有先用插件定位 clipboard、hashdump、hivelist、cmdline、netscan。

## 关联技巧

- [cross-domain-forensics-technique-map.md](cross-domain-forensics-technique-map.md)
- [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)
- [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md)
- [linux-git-browser-and-container-forensics.md](linux-git-browser-and-container-forensics.md)
- [network-covert-auth-and-reassembly.md](network-covert-auth-and-reassembly.md)
- [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md)
- [forensics-tooling.md](forensics-tooling.md)

## 原始资料

- [windows-registry-logs-and-credentials.md](../raw/forensics/windows-registry-logs-and-credentials.md)
