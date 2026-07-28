---
type: technique
tags: [reverse, loader, staged-payload, memory-image, unpacking]
skills: [ctf-reverse, ctf-malware]
raw:
  - ../raw/reverse/loader-vm-image-and-kernel-patterns.md
  - ../raw/reverse/packers-deobfuscation-and-debug-automation.md
  - ../raw/reverse/Spirit2026-5-link-start-wp.md
updated: 2026-07-28
---

# Staged Loader and Runtime-Image Recovery

## 适用场景

首阶段程序只负责解密、映射、重定位或下载第二阶段；静态文件不是最终分析对象，需要在映射完成、控制权转移前后 dump 真实代码和运行时依赖。若第二阶段由握手、路径或第一段输入解密，同样应先固定触发条件，而不是完整反虚拟化外壳。

## 识别信号

- 大量 `VirtualAlloc/mmap/mprotect`、解压/解密后间接跳转。
- 文件节区熵高、入口逻辑短，运行后出现新的可执行内存。
- 自定义 PE/ELF/bytecode loader 或资源中携带第二镜像。
- client/server 两端或握手输入共同决定 SMC 解密窗口，运行后内存才出现正常函数序言。

## 最小证据

- 找到写入可执行内存和最终 transfer-of-control 的位置。
- 保存映射基址、长度、权限、解密后 hash 和入口地址。
- Dump 后镜像可由反汇编器解析或能修复到可运行。
- 若映像依赖输入，记录触发 payload、通信 framing、key 派生和 dump 时刻。

## 解法骨架

1. 跟踪分配、写入、解密和权限变化。
2. 在重定位/import 修复完成且跳转前后分别 dump。
3. 按映射表修复节区、IAT/relocation 或直接以 memory layout 加载分析器。
4. 对输入条件化 SMC，先重放最短握手/路径输入，再在代码可执行且尚未覆盖时 dump。
5. 对第二阶段重新做字符串、控制流和输入验证分析；RC4/AES 等标准算法转相应 crypto technique。

## 关键变体

| 变体 | 关键证据 | 处理 |
|---|---|---|
| Userland PE/ELF manual map | 自行分配、重定位并修复 imports | 在映射和 import 完成后 dump。 |
| 资源/网络解密第二阶段 | blob 只在内存短暂明文存在 | 盯分配、写入、权限切换与间接跳转。 |
| 自定义 bytecode/VM image loader | 载入的是 VM 状态而非原生映像 | 保存 image、入口和 handler 映射。 |
| 输入条件化 SMC/client-server | 第一段输入或对端消息兼作解密 key | 固定握手并记录解密窗口，不必完整理解 VMP。 |

## 常见陷阱

- Dump 时机太早，内容仍加密或未重定位。
- 只保存单一区域，漏掉分离的数据/导入表。
- 把 loader 的校验逻辑误当成最终算法。

## 关联技巧

- [loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md)
- [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md)
- [trace-hook-and-state-snapshot-reconstruction.md](trace-hook-and-state-snapshot-reconstruction.md)
- [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)
- [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md)

## 原始资料

- [loader-vm-image-and-kernel-patterns.md](../raw/reverse/loader-vm-image-and-kernel-patterns.md)
- [packers-deobfuscation-and-debug-automation.md](../raw/reverse/packers-deobfuscation-and-debug-automation.md)
- [Spirit2026-5-link-start-wp](../raw/reverse/Spirit2026-5-link-start-wp.md)
