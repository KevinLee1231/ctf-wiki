---
type: technique
tags: [forensics, memory, process, container, layer]
skills: [ctf-forensics, ctf-cloud-infra]
raw:
  - ../raw/forensics/disk-memory-vm-and-container-forensics.md
  - ../raw/forensics/filesystems-memory-dumps-and-raid.md
updated: 2026-07-27
---

# Memory, Process and Container-Layer Recovery

## 适用场景

内存转储、VM snapshot、容器层或 overlay filesystem 中需要恢复进程、命令、注入代码、环境变量、删除文件或层间差异。

## 识别信号

- LiME/raw/minidump/VMDK memory、Docker/OCI layer、overlay upperdir。
- 目标数据只存在于进程内存、环境、打开文件或被后续层删除。
- 需要从进程/VAD/module/socket 与容器 manifest/layer whiteout 关联。

## 最小证据

- 确认 OS/kernel/profile、采集时间、镜像 hash 和容器层顺序。
- 每个恢复对象记录进程/PID/VAD 或 layer/diff path。
- Dump 结果有 PE/ELF/文本/配置结构校验。

## 解法骨架

1. 识别镜像类型、OS/架构和采集上下文。
2. 内存侧枚举进程、模块、VAD、句柄、命令和网络；容器侧按 manifest 叠加 layer。
3. Dump 可疑区域/文件，处理 deleted mapping、whiteout 和 overlay precedence。
4. 将进程时间、文件层和网络证据关联到同一行为链。

## 关键变体

- Process/VAD injected code recovery。
- VM snapshot/minidump artifact extraction。
- Docker/OCI layer 与 whiteout 恢复。

## 常见陷阱

- 使用错误 profile/符号，解析结果看似正常却字段错位。
- 只看容器最终 rootfs，漏掉下层已删除 secret。
- Dump 内存字符串后未关联具体进程和地址。

## 关联技巧

- [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)
- [filesystems-memory-dumps-and-raid.md](filesystems-memory-dumps-and-raid.md)
- [windows-registry-event-and-credential-correlation.md](windows-registry-event-and-credential-correlation.md)

## 原始资料

- [disk-memory-vm-and-container-forensics.md](../raw/forensics/disk-memory-vm-and-container-forensics.md)
- [filesystems-memory-dumps-and-raid.md](../raw/forensics/filesystems-memory-dumps-and-raid.md)
