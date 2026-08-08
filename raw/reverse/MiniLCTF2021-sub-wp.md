# MiniLCTF 2021 - sub

## 题目简述

附件是带自定义壳的 32 位 Windows PE。外层程序把一个按字节异或 `0x64` 的 PE 追加在新节中，运行时通过进程镂空把它映射到傀儡进程。内层 PE 还在 TLS 回调中检查调试端口：检测到调试器时把函数指针指向假校验，正常运行时才指向真校验。因此直接跟进显眼的检查函数只能得到诱饵 flag。

## 解题过程

节表和运行行为给出两个脱壳入口：

1. 静态方式：定位最后一个附加节，对节数据逐字节异或 `0x64`，按嵌入 PE 的 `MZ`/`PE` 结构重新导出；
2. 动态方式：在 `WriteProcessMemory` 完成映射后、`ResumeThread` 前暂停，从傀儡进程的映像基址转储并修复导入表。

内层程序有一个全局函数指针 `check`，初始指向 `fake_check`。TLS 回调调用 `NtQueryInformationProcess(ProcessDebugPort)`；调试状态下继续使用假函数，非调试状态下改为 `correct_check`。假函数满足：

```text
tmp = ((input[i] ^ 0x66) + 4) ^ 0x55
```

逆运算后只得到诱饵：

```text
miniLctf{Th1s_1s_th4_fak4_f1ag!}
```

真正数组与算法为：

```python
correct_enc = [
    0x5A, 0x26, 0x59, 0x26, 0x7B, 0x5C, 0x43, 0x51,
    0x54, 0x6D, 0x52, 0x68, 0x0E, 0x4C, 0x68, 0x4C,
    0x0F, 0x68, 0x0E, 0x59, 0x43, 0x03, 0x4D, 0x03,
    0x4C, 0x43, 0x0E, 0x59, 0x50, 0x1E, 0x1E, 0x4A,
]

flag = "".join(chr(((x ^ 0x66) - 4) ^ 0x55) for x in correct_enc)
print(flag)
```

得到：

```text
miniLctf{Re_1s_s0_1nt4r4st1ng!!}
```

调试时可以直接把 `check` 函数指针改到真函数，或让 `ProcessDebugPort` 返回 0；但最终仍应从真函数的变换和常量数组独立恢复输入，而不是把内存中出现的字符串当答案。

## 方法总结

该题有三层干扰：附加节异或、进程镂空、TLS 反调试选取假校验。分析顺序应是先恢复真实执行映像，再枚举入口点之前的 TLS 回调，最后追踪间接调用的实际目标。只解出一个格式正确的字符串并不等于完成，必须确认它能通过非调试路径选择的真校验函数。
