---
type: technique
tags: [forensics, windows, registry, event-log, credentials]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/windows-registry-logs-and-credentials.md
  - ../raw/forensics/SUCTF2026-forensicsWP.md
updated: 2026-07-27
---

# Windows Registry, Event and Credential Correlation

## 适用场景

Windows 镜像/事件包中需要把 Registry hive、EVTX、NTFS、浏览器/DPAPI 凭据和用户活动按 SID、时间与主机状态关联，恢复登录、执行或秘密使用链。

## 识别信号

- `SYSTEM/SAM/SECURITY/SOFTWARE/NTUSER.DAT`、EVTX、Prefetch、LNK、浏览器 profile。
- 需要 boot key、用户 SID、DPAPI masterkey 或登录事件解密凭据。
- 文件时间与 Registry/EventLog 记录存在时区或 anti-forensics 差异。

## 最小证据

- 固定镜像时区、主机名、SID 和关键 hive ControlSet。
- 每个事件记录 source、event id、record id 和原时间字段。
- 凭据/解密结果与用户、主机和使用行为对应。

## 解法骨架

1. 从 hive 恢复系统信息、用户、时区、网络和持久化配置。
2. 解析 EVTX/NTFS/Prefetch/LNK 建统一时间线。
3. 按 SID 和登录上下文关联浏览器、DPAPI、SAM/LSA 证据。
4. 用后续登录、访问或解密对象验证恢复秘密。

## 关键变体

- Registry/EVTX 时间线。
- SAM/LSA/DPAPI/浏览器凭据。
- NTFS USN/$LogFile 与反取证时间戳。

## 常见陷阱

- 混用本地时间和 UTC。
- 使用错误 ControlSet 或用户 SID。
- 找到 credential blob 但未完成 masterkey/上下文解密。

## 关联技巧

- [windows-registry-logs-and-credentials.md](windows-registry-logs-and-credentials.md)
- [filesystem-metadata-and-deleted-artifact-recovery.md](filesystem-metadata-and-deleted-artifact-recovery.md)
- [memory-process-and-container-layer-recovery.md](memory-process-and-container-layer-recovery.md)

## 原始资料

- [windows-registry-logs-and-credentials.md](../raw/forensics/windows-registry-logs-and-credentials.md)
- [SUCTF2026-forensicsWP](../raw/forensics/SUCTF2026-forensicsWP.md)
