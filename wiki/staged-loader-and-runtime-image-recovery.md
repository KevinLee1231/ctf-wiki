---
type: technique
tags: [reverse, loader, staged-payload, memory-image, unpacking]
skills: [ctf-reverse, ctf-malware]
raw:
  - ../raw/reverse/loader-vm-image-and-kernel-patterns.md
  - ../raw/reverse/packers-deobfuscation-and-debug-automation.md
updated: 2026-07-27
---

# Staged Loader and Runtime-Image Recovery

## 适用场景

首阶段程序只负责解密、映射、重定位或下载第二阶段；静态文件不是最终分析对象，需要在映射完成、控制权转移前后 dump 真实代码和运行时依赖。

## 识别信号

- 大量 `VirtualAlloc/mmap/mprotect`、解压/解密后间接跳转。
- 文件节区熵高、入口逻辑短，运行后出现新的可执行内存。
- 自定义 PE/ELF/bytecode loader 或资源中携带第二镜像。

## 最小证据

- 找到写入可执行内存和最终 transfer-of-control 的位置。
- 保存映射基址、长度、权限、解密后 hash 和入口地址。
- Dump 后镜像可由反汇编器解析或能修复到可运行。

## 解法骨架

1. 跟踪分配、写入、解密和权限变化。
2. 在重定位/import 修复完成且跳转前后分别 dump。
3. 按映射表修复节区、IAT/relocation 或直接以 memory layout 加载分析器。
4. 对第二阶段重新做字符串、控制流和输入验证分析。

## 关键变体

- Userland PE/ELF manual map。
- 资源/网络解密第二阶段。
- 自定义 bytecode/VM image loader。

## 常见陷阱

- Dump 时机太早，内容仍加密或未重定位。
- 只保存单一区域，漏掉分离的数据/导入表。
- 把 loader 的校验逻辑误当成最终算法。

## 关联技巧

- [loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md)
- [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md)
- [trace-hook-and-state-snapshot-reconstruction.md](trace-hook-and-state-snapshot-reconstruction.md)

## 原始资料

- [loader-vm-image-and-kernel-patterns.md](../raw/reverse/loader-vm-image-and-kernel-patterns.md)
- [packers-deobfuscation-and-debug-automation.md](../raw/reverse/packers-deobfuscation-and-debug-automation.md)
