---
type: technique
tags: [reverse, crypto, technique, python, pyd, cython, loader, aes, xxtea]
skills: [ctf-reverse, ctf-crypto]
raw:
  - ../raw/reverse/Spirit2026-5-cythonchecker-wp.md
updated: 2026-05-21
---

# Embedded Python PYD Loader and Custom AES Checker

## 适用场景

Windows PE 外壳不直接校验 flag，而是在运行时恢复 key、释放 `.pyd` 扩展模块、初始化嵌入式 Python，然后调用模块中暴露的 `verify_flag(flag, key_bytes)`。真正校验逻辑藏在 Cython / CPython extension 里，可能结合标准密码结构的微小改动。

## 识别信号

- 外层 PE 字符串或导入表出现 `python312.dll`、`PyImport_ImportModule`、`PyObject_GetAttrString`、`PyUnicode_FromString`、`PyBytes_FromStringAndSize` 等嵌入式 Python API。
- 程序运行时写出 `runtime_codec.pyd`、`*.pyd` 或临时目录下的 Python 扩展模块。
- 外层恢复两类 key：一类用于解包 `.pyd`，一类传入 `verify_flag` 作为真实校验 key。
- `.pyd` 中可见 `_sub_bytes`、`_shift_rows`、`_mix_columns`、`_expand_round_keys`、S-box 或 round key，结构高度类似 AES。

## 最小证据

- 已定位外壳中恢复 key 的函数，并能导出传入校验函数的 key。
- 已拿到运行时实际释放的 `.pyd`，而不是工程目录里可能过期的旧构建。
- 已确认 `verify_flag` 的参数约束、payload 提取方式、padding 方式和目标密文比较点。
- 已识别密码结构中与标准 AES 的差异，例如 `ShiftRows = (0, 1, 3, 2)`。

## 解法骨架

1. 先分析 loader：确认它是否只是壳，找出恢复 key、释放 `.pyd`、初始化 Python、调用 `verify_flag` 的主流程。
2. 恢复外层 key：通常是线性 XOR、滚动异或或轻量解密；分清“解包 key”和“校验 key”。
3. 获取真实 `.pyd`：优先运行一次程序从临时目录复制；需要完整静态复现时，再用解包 key 解密内嵌 blob。
4. 逆 `.pyd`：从 `PyInit_<module>`、导出函数表和 `verify_flag` wrapper 进入真正的 C/Cython 逻辑。
5. 复原校验：确认 flag wrapper、payload 长度、字符集、padding、目标密文和加密模式。
6. 如果算法像 AES，先对比 S-box、MixColumns、KeySchedule 和 ShiftRows；只修改变体点。
7. 用目标密文、恢复出的 key 和变体逆轮离线解密 payload，再包回 flag 格式。

## 关键变体

- **运行时释放的 `.pyd` 是唯一可信样本。** 目录中的旧构建文件可能与当前 loader key 或 target ciphertext 不匹配。
- **Cython wrapper 与真实函数分层。** wrapper 负责类型检查、参数转换和异常处理；真正算法常在内部 helper 中。
- **标准密码的单点变体。** 看到 AES 结构时，优先验证哪个步骤被改动；常见变体是行移位、轮常量、S-box 或轮数。
- **动态断点提取目标密文。** 在 `PyBytes_FromStringAndSize`、目标 bytes 构造点或比较点下断，比盲猜静态常量更稳。

## 常见陷阱

- 误用错误 `.pyd` 样本会导致 key、target ciphertext 和解密结果全部对不上。
- 看到 Python API 就按 Python bytecode 处理；`.pyd` 是原生扩展，仍然需要原生逆向。
- 把整个算法当自定义密码重建，忽略它其实是 AES-128 的小变体。
- 只分析 loader 不分析释放出的模块，会错过真正的 flag 校验逻辑。

## 关联技巧

- [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md)
- [python-bytecode-esolangs-and-uefi.md](python-bytecode-esolangs-and-uefi.md)
- [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md)
- [block-mode-misuse-family.md](block-mode-misuse-family.md)
- [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)

## 原始资料

- [Spirit2026-5-cythonchecker-wp.md](../raw/reverse/Spirit2026-5-cythonchecker-wp.md)
