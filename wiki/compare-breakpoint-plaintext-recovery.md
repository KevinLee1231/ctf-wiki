---
type: technique
tags: [reverse, crypto, dynamic-debugging, breakpoint, symmetric-checker, xxtea]
skills: [ctf-reverse, ctf-crypto]
raw:
  - ../raw/reverse/Spirit2026-5-xxxtea-wp.md
updated: 2026-07-06
---

# Compare Breakpoint Plaintext Recovery

## 适用场景

校验函数看起来包含复杂加密、XXTEA 或自定义对称算法，但程序实际先把内置数据解密成明文 flag，再和输入比较。此时最短路径不是完整逆算法，而是在最终比较点或解密后缓冲区断下直接读取明文。

## 识别信号

- 加密流程很长，但输入在关键循环中没有明显参与变换。
- 最终比较前出现一个长度接近 flag 的局部缓冲区。
- 输入只用于长度检查和最终 `strcmp` / `memcmp` / 字节比较。
- 动态调试到比较点时，某个变量已经包含可读 flag。

## 最小证据

- 已确认输入需要满足固定长度才能进入完整校验流程。
- 已定位最终比较或失败跳转附近的断点。
- 已观察到解密后明文缓冲区与 flag 格式匹配。

## 解法骨架

1. 用占位输入满足长度检查，不要因为长度错误错过关键路径。
2. 静态找最终比较点：`strcmp`、`memcmp`、逐字节循环、成功/失败分支、flag format 常量引用。
3. 动态调试到比较前，分别观察“用户输入 buffer”和“程序内部 buffer”；对可疑局部变量、堆 buffer 和 `.data` 解密区下硬件/内存断点。
4. 如果内部 buffer 已是 `flag{...}` 或明显明文，直接 dump，并回看数据流确认输入没有参与前置解密。
5. 若明文只短暂出现，改用 trace / Frida hook / x64dbg log 在比较函数入口记录参数。
6. 只有当比较点前没有可读明文时，才投入 XXTEA 或自定义算法复现。

## 关键变体

- **解密后比较。** 方向是内置密文到明文，而不是输入加密到密文。
- **长度门槛。** 不满足长度时到不了关键断点。
- **局部变量命名误导。** `v7`、`buf`、`dest` 这类名字没有语义，依据应是运行时内容。
- **PDF 截图依赖。** 关键变量位置可能只在图片里，需要看提取图片。

## 常见陷阱

- 被 XXTEA 名称或复杂循环吓到，直接投入算法还原。
- 只看输入数据流，没有看内置密文和解密结果生命周期。
- 输入长度不对导致断点永远不触发。
- 只在 `main` 里找 flag，忽略最终比较函数参数和短生命周期局部缓冲区。
- 动态调试得到明文后没有正向复核，容易把中间测试串误判为 flag。

## 关联技巧

- [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md)
- [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)
- [frida-angr-lldb-and-x64dbg.md](frida-angr-lldb-and-x64dbg.md)
- [block-mode-misuse-family.md](block-mode-misuse-family.md)
- [embedded-python-pyd-custom-aes.md](embedded-python-pyd-custom-aes.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [Spirit2026-5-xxxtea-wp](../raw/reverse/Spirit2026-5-xxxtea-wp.md) | 复杂 XXXTEA 外观下程序先解出明文再比较，长度门槛满足后断在最终比较点即可。 |

## 原始资料

- [Spirit2026-5-xxxtea-wp.md](../raw/reverse/Spirit2026-5-xxxtea-wp.md)
