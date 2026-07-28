---
type: technique
tags: [reverse, python, cython, pyd, extension-module, checker]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/Spirit2026-5-cythonchecker-wp.md
  - ../raw/reverse/D3CTF2022-d3thon-wp.md
  - ../raw/reverse/D3CTF2023-d3recover-wp.md
updated: 2026-07-28
---

# Cython and Python-Extension Checker Recovery

## 适用场景

Python 逻辑被编译为 Cython `.so`/`.pyd` 或由原生 loader 运行时释放并调用；需要从模块初始化、字符串表、`_Pyx_*` API、对应有符号版本或动态导入行为中恢复解释器/校验语义。

## 识别信号

- 二进制导出 `PyInit_<module>`，或大量出现 `_Pyx_*`、`PyObject_*`、`PyNumber_*` API。
- loader 调用 `PyImport_ImportModule`、`PyObject_GetAttrString` 并释放临时 `.pyd`/`.so`。
- Cython 模板代码与异常处理噪声占比很高，但字符串表保留 Python 名称。
- 同题提供 stripped/unstripped 或相近版本，可用 BinDiff/符号迁移定位 checker。

## 最小证据

- 拿到运行时实际加载的扩展模块，并记录 hash、Python ABI 与模块名。
- 从 `PyInit_*` 或模块方法表定位目标导出函数及参数类型。
- 至少把一组 `_Pyx_*`/CPython API 调用还原成 Python 层容器、算术或比较语义。
- 候选输入能通过原模块或独立 forward checker 验证。

## 解法骨架

1. 先区分原生 loader、Python wrapper 与扩展模块，确认真正校验位于哪一层。
2. 动态释放时在文件写入、模块加载或 `PyInit_*` 前后 dump 实际模块。
3. 从全局字符串表、方法定义表和 `_pyx_pymod_exec_*` 定位高语义函数。
4. 把 `_Pyx_PyObject_GetItem`、`PyObject_SetItem`、`PyNumber_*` 等映射回 Python 操作。
5. 有相近有符号版本时先做函数匹配，再把 checker 翻译成逆运算、SMT 或脚本。
6. 若内部是标准密码变体，独立确认 S-box、轮函数、模式和单点差异，不把密码细节写死在 Cython 路由中。

## 关键变体

| 变体 | 优先路线 |
|---|---|
| Loader 运行时释放 `.pyd` | 先 dump 真实模块并区分解包 key 与校验 key。 |
| Cython 字符串表完整 | 从 `_pyx_n_s_*`/`_pyx_k_*` 名称定位主逻辑。 |
| stripped 与 unstripped 双版本 | BinDiff/符号迁移后恢复 checker。 |
| 模块可被直接 import | 用最小输入逐条探测 API/bytecode 语义。 |
| 内部是 AES/XXTEA 等变体 | 先恢复 Python/Cython wrapper，再转相应密码 technique。 |

## 常见陷阱

- 看到 Python API 就按 `.pyc` 处理；`.pyd`/`.so` 仍是原生扩展。
- 使用工程目录中的旧构建，而不是 loader 实际释放的模块。
- 从 Cython 模板异常路径逐行分析，忽略字符串表和模块方法表。
- 把某一题的 AES ShiftRows 变体当成所有 Cython checker 的共同模式。

## 关联技巧

- [managed-runtime-metadata-and-bytecode-recovery.md](managed-runtime-metadata-and-bytecode-recovery.md)
- [staged-loader-and-runtime-image-recovery.md](staged-loader-and-runtime-image-recovery.md)
- [constraint-solver-and-symbolic-state-recovery.md](constraint-solver-and-symbolic-state-recovery.md)
- [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)
- [block-mode-misuse-family.md](block-mode-misuse-family.md)

## 原始资料

- [Spirit2026-5-cythonchecker-wp](../raw/reverse/Spirit2026-5-cythonchecker-wp.md)
- [D3CTF2022-d3thon-wp](../raw/reverse/D3CTF2022-d3thon-wp.md)
- [D3CTF2023-d3recover-wp](../raw/reverse/D3CTF2023-d3recover-wp.md)
