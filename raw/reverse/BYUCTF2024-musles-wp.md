# Musles

## 题目简述

程序使用 musl，先 `mmap` 一段 RWX 内存，把全局 `payload` 复制进去并逐字节异或 `0x20`，然后执行。它还通过 `pidof gdb` 检测调试器。真正的 flag 校验位于运行时解密出的代码中。

## 解题过程

无需先绕过反调试就能静态恢复第二阶段：

```python
decoded = bytes(x ^ 0x20 for x in payload)
open("stage2.bin", "wb").write(decoded)
```

把 `stage2.bin` 按 x86-64 反汇编。它先从 stdin 向栈读取 38 字节，随后以 8 字节为组做 XOR/比较。对于形如

```asm
xor qword ptr [rsp], rcx
cmp qword ptr [rsp], rax
```

的检查，原输入块等于 `rax ^ rcx`；注意 x86-64 立即数在内存中按 little-endian 解释。依次恢复得到：

```text
byuctf{u
r_GDB_sk
ills_are
_really_
swoll}
```

拼接为：

```text
byuctf{ur_GDB_skills_are_really_swoll}
```

动态验证时可在 `pidof gdb` 分支后附加调试器，或修改条件跳转；但 flag 恢复并不依赖运行程序。

## 方法总结

自修改/运行时解密代码应先分层：第一层只负责映射、反调试和 XOR，第二层才包含校验逻辑。对每层单独导出并反汇编，比在混合控制流中硬跟踪更清晰；常量恢复还必须遵守目标端序。
