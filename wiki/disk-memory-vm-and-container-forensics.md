---
type: family
tags: [forensics, family, disk, memory, vm, container, cloud]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/disk-memory-vm-and-container-forensics.md
  - ../raw/forensics/filesystems-memory-dumps-and-raid.md
updated: 2026-07-28
---

# Disk, Memory, VM and Container Forensics

## 作用边界

本页是磁盘、内存、虚拟机、容器和云存储取证 family。它同时负责载体选择和底层恢复分流：先判断证据源是完整镜像、内存快照、VM/快照、core dump、Android 备份、Docker layer、云 bucket、RAID/分卷还是数据库碎片，再进入相应 technique。

它与 [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md) 的边界在于：本页处理载体、分区、RAID、VMDK、内存和容器层；后者处理已经进入单个文件/归档后的 header、压缩流、ZipCrypto 和删除对象恢复。

## 识别信号

- 附件是 `.raw`、`.img`、`.dd`、`.vmdk`、`.ova`、`.vmem`、`.vmss`、core dump、Android backup、Docker image/layer、KAPE collection 或云存储导出。
- flag 藏在历史状态、已删除文件、环境变量、进程内存、容器层、构建命令、数据库碎片或勒索脚本 key 中。
- 需要只读挂载、carving、Volatility/MemProcFS、layer diff、timeline、strings、registry/Amcache/MFT 或数据库解析。

## 最小证据

- 识别容器格式和层级：磁盘分区、文件系统、VM 容器、内存类型、Docker layer、Android backup 或云对象。
- 记录只读处理方式和导出的中间文件，避免污染证据。
- 明确搜索目标：文件内容、凭据、key、进程、环境变量、网络痕迹、时间线或删除历史。
- 能解释从载体到最终 artifact 的恢复路径。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| 完整磁盘镜像、分区表、删除文件 | 先只读挂载和分区识别，再用 metadata/journal/unallocated block 恢复 | [filesystem-metadata-and-deleted-artifact-recovery.md](filesystem-metadata-and-deleted-artifact-recovery.md) |
| 可挂载文件系统内的删除文件、损坏压缩包、ZipCrypto | 载体已经清楚，重点是文件/归档层恢复 | [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md) |
| 内存 dump、VM snapshot、core dump | 先判断 OS/进程/地址空间，再恢复环境变量、映射文件、key 或运行时对象 | [memory-process-and-container-layer-recovery.md](memory-process-and-container-layer-recovery.md) |
| OVA/VMDK/VMDK sparse、VMware snapshot | 先展开容器并识别 sparse grain、snapshot chain 和磁盘/内存组件 | [raid-vm-disk-and-fragmented-volume-reconstruction.md](raid-vm-disk-and-fragmented-volume-reconstruction.md) |
| RAID5/XOR、缺盘、分卷或损坏 partition | 先确定成员顺序、chunk size、parity 方向和几何映射 | [raid-vm-disk-and-fragmented-volume-reconstruction.md](raid-vm-disk-and-fragmented-volume-reconstruction.md) |
| Docker image/layer、container diff、history | 先看 layer.tar、whiteout、history、ENV/ARG 和删除文件 | [linux-git-browser-and-container-forensics.md](linux-git-browser-and-container-forensics.md) |
| Android backup/APK/data 分区 | 先分离 APK、shared_prefs、SQLite、WiFi/浏览器/应用数据 | [mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md) |
| 云 bucket/versioning 导出 | 先查公开权限、版本历史、对象元数据和删除版本 | [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md) |
| PowerShell 勒索/恶意脚本内存残留 | 先提取脚本和 key，再转 malware 或 crypto 解密 | [scripts-and-obfuscation.md](scripts-and-obfuscation.md), [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md) |
| TrueCrypt/VeraCrypt 高熵卷或 keyfile | 先确认卷头、上下文密码/keyfile 和隐藏卷可能性，再进入卷内恢复 | [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md) |
| SQLite/Kyoto Cabinet/BSON/diff history | 先解析记录结构和顺序，再重放历史内容 | [linux-git-browser-and-container-forensics.md](linux-git-browser-and-container-forensics.md) |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| 文件系统 metadata、journal 或未分配块可恢复删除对象 | [filesystem-metadata-and-deleted-artifact-recovery.md](filesystem-metadata-and-deleted-artifact-recovery.md) |
| 进程内存、VM snapshot 或 OCI layer 保留运行时/历史对象 | [memory-process-and-container-layer-recovery.md](memory-process-and-container-layer-recovery.md) |
| RAID/VMDK/分卷/损坏分区必须先恢复几何和映射 | [raid-vm-disk-and-fragmented-volume-reconstruction.md](raid-vm-disk-and-fragmented-volume-reconstruction.md) |

## 合并与拆分结论

原 `filesystems-memory-dumps-and-raid.md` 已并入本页：两页的三个 technique 下一跳完全相同，额外中转不能改变路线。现在由本页直接区分 metadata 恢复、运行时对象恢复和 RAID/VM 映射；只有文件/归档层问题再转相邻 family。

## 常见陷阱

- 直接挂载写入镜像，破坏时间戳或恢复状态。
- 只跑 strings，没先识别文件系统和快照层级。
- Docker 题只看最终层，忽略历史层和 whiteout 删除痕迹。
- 内存题没确认 OS/profile，插件输出误导。
- 云存储题只看当前对象，没查 versioning 和 metadata。

## 关联技巧

- [cross-domain-forensics-technique-map.md](cross-domain-forensics-technique-map.md)
- [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md)
- [raid-vm-disk-and-fragmented-volume-reconstruction.md](raid-vm-disk-and-fragmented-volume-reconstruction.md)
- [linux-git-browser-and-container-forensics.md](linux-git-browser-and-container-forensics.md)
- [windows-registry-logs-and-credentials.md](windows-registry-logs-and-credentials.md)
- [forensics-tooling.md](forensics-tooling.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [D3CTF2019-vera-wp](../raw/forensics/D3CTF2019-vera-wp.md) | VeraCrypt 容器和图像光栅恢复组合，先用题面线索找密码再处理解出的媒体 artifact。 |
| [D3CTF2021-virtual-love-revenge-2-0-wp](../raw/forensics/D3CTF2021-virtual-love-revenge-2-0-wp.md) | VMware 配置损坏、零宽字符字典和单用户模式取证组合，先修复虚拟机再进入磁盘证据链。 |
| [D3CTF2022-wannawacca-wp](../raw/malware/D3CTF2022-wannawacca-wp.md) | Windows 内存镜像、勒索样本和流量协议复现组合，先 dump 可疑进程和文件再还原加密链。 |
| [D3CTF2019-disappeared-memory-wp](../raw/reverse/D3CTF2019-disappeared-memory-wp.md) | Windows 10 compressed memory 导致关键页缺失，先从 dump/PTE/Store Manager 恢复数据页。 |
| [0xGame2021-week4-Strobe-Memory-wp](../raw/forensics/0xGame2021-week4-Strobe-Memory-wp.md) | 本题的关键证据链是“识别系统 profile → 恢复口令 `ffxiv` → 从命令行历史定位密文 → 按题面用 RC4 解密 → 将大整数解释为 Tupper 位图”。`U2FsdGVkX1` 不是算法指纹；只有把内存取证得到的口令、题面给出的算法和解密后的绘图参数串联起来，结论才可复现。 |
| [0xGame2022-week4-ColorfulDisk-wp](../raw/forensics/0xGame2022-week4-ColorfulDisk-wp.md) | 完整证据链是“磁盘镜像 → 加密容器 → PNG 像素字节 → WAV → SSTV 图像 → 压缩包”。遇到看似噪声的图片时，应结合尺寸、提示词和导出字节的文件头判断数据组织方式；这里必须保留 RGB 通道顺序和行优先顺序，否则无法还原可识别的 WAV。 |
| [0xGame2023-week4-oh-my-linux-wp](../raw/forensics/0xGame2023-week4-oh-my-linux-wp.md) | Linux 内存取证首先要精确匹配内核 Profile，随后再围绕进程历史建立证据链。本题中 `.zsh_history` 不是 flag 本身，却把三个载体、压缩密码来源和别名入口串联起来；恢复文件后应按命令发生顺序解释每条线索，最后再拼接片段。 |
| [0xGame2024-week3-画画的baby-wp](../raw/forensics/0xGame2024-week3-画画的baby-wp.md) | 本题恢复的是进程内存中的绘图表面，而不是磁盘上的图片文件。分析 GUI 应用内存时，应先用进程列表确定正确 PID，再转储地址空间并按原始像素数据尝试宽度、色彩通道和偏移；同时要注意 Volatility 输出中的 PID 与 PPID 列，避免转储错误进程。 |
| [UMDCTF2021-philip-1-wp](../raw/forensics/UMDCTF2021-philip-1-wp.md) | Linux 内存取证要把“行为证据”和“文件证据”串起来：`linux_psaux`、`linux_bash` 说明用户做过什么，`linux_enumerate_files` 与 `linux_find_file` 才负责恢复实际凭据。提取 SSH 私钥后还要修正权限，否则 OpenSSH 会拒绝使用。 |
| [UMDCTF2021-philip-3-wp](../raw/forensics/UMDCTF2021-philip-3-wp.md) | 内存中的逻辑顺序不一定等于物理连续顺序。源码给出了节点布局和遍历关系，进程映射给出地址基准，堆转储提供实际字节；三者结合后才能正确跟随指针。直接对镜像跑 `strings` 可能只看到零散字符，无法证明顺序。 |
| [UMDCTF2022-kernel-infernal-2-wp](../raw/forensics/UMDCTF2022-kernel-infernal-2-wp.md) | 内存取证中必须区分硬件寄存器值、物理地址和内核虚拟指针。若题目真正要求可写回 CR3 的物理基址，还需把 `pgd` 虚拟地址转换为物理地址并处理低位标志；本题的判题值直接采用 `mm_struct.pgd`。遍历任务时还要跳过 `mm == NULL` 的内核线程。 |
| [WMCTF2023-fantastic-terminal-wp](../raw/forensics/WMCTF2023-fantastic-terminal-wp.md) | 对浏览器终端题同时检查通信编码和浏览器进程内存残留；终端输入输出中出现大段 base64，或题目把完整运行环境搬到浏览器内时，应考虑解码传输数据和 dump 对应标签页进程。 |
| [WMCTF2023-truncate-wp](../raw/forensics/WMCTF2023-truncate-wp.md) | 用 Volatility 恢复 Linux 内存中的命令历史和文件系统，再结合输入事件绘图、PNG 尾部残留恢复和 AES 解密拼接 flag；内存镜像中出现 remmina、event2、base64 图片和被截断 PNG 时，应联想到鼠标轨迹恢复与 Acropalypse 类残留数据恢复。 |

## 原始资料

- [disk-memory-vm-and-container-forensics.md](../raw/forensics/disk-memory-vm-and-container-forensics.md)
- [filesystems-memory-dumps-and-raid.md](../raw/forensics/filesystems-memory-dumps-and-raid.md)
