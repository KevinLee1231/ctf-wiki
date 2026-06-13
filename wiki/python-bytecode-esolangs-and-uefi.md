---
type: family
tags: [reverse, family, python-bytecode, esolang, uefi, bytecode]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/python-bytecode-esolangs-and-uefi.md
updated: 2026-06-12
---

# Python Bytecode, Esolangs and UEFI

## 作用边界

本页是非标准字节码与低频执行格式 family，覆盖 Python `.pyc`/opcode remap/Pyarmor/Nuitka/Cython 相关入口、Brainfuck/FRACTRAN/函数式语言、HarmonyOS ABC、UEFI、DOS stub、coverage side-channel 和其它解释器式载体。它不再作为单一 technique。

## 首轮路由

| 信号 | 先确认 | 下一跳 |
|---|---|---|
| `.pyc`、`dis` 输出、magic/version、opcode remap | Python 版本、字节码版本、opcode 表、常量表和控制流是否可还原 | [embedded-python-pyd-custom-aes.md](embedded-python-pyd-custom-aes.md)、[disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md) |
| Pyarmor/Nuitka/Cython/嵌入式 Python | 入口是 Python 层、native 扩展、模块 stub 还是运行时 dump | [embedded-python-pyd-custom-aes.md](embedded-python-pyd-custom-aes.md)、[runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md) |
| Brainfuck/FRACTRAN/OPAL/函数式语言 | 是否可反向解释、转 C、抽约束或用 coverage/read count oracle | [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md) |
| HarmonyOS HAP/ABC、UEFI、DOS stub | 先恢复格式、工具链、入口点和调用约定，再进入算法恢复 | [mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md)、[hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md) |
| Unity IL2CPP、游戏字节码、非原生脚本 | 先确认 metadata、managed/native 边界和资源脚本关系 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md) |

## 合并与拆分结论

- 保留为 family：这些 raw 分支的共同障碍是“先恢复执行格式/字节码语义”，不是某个具体算法。
- 不合并进 VM family：VM family 判断自定义 VM/混淆，本页承接 Python/esolang/UEFI/HarmonyOS 等已有格式或小众解释器。
- 暂不拆 Python 字节码 technique：已有 raw/WP 覆盖 `.pyc`、Cython、Pyarmor、Nuitka 等不同入口，先用 family 分流更稳。

## 常见误判

- 忽略 Python 版本，导致用错 opcode 表或 `.pyc` magic。
- 把 Pyarmor/Nuitka/Cython 都当成普通 `.pyc`，没有检查 native runtime 和模块导入边界。
- Esolang 题直接手工模拟全程序，没先找输入读取点、比较 idiom、反向解释或 side-channel。
- UEFI/HarmonyOS 题未先确认格式和工具链，就强行套普通 PE/ELF 分析。

## 关联页面

- [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md)
- [embedded-python-pyd-custom-aes.md](embedded-python-pyd-custom-aes.md)
- [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md)
- [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)
- [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)
- [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md)

## 原始资料

- [python-bytecode-esolangs-and-uefi.md](../raw/reverse/python-bytecode-esolangs-and-uefi.md)
