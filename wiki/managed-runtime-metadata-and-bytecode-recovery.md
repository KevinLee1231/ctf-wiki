---
type: technique
tags: [reverse, bytecode, dotnet, jvm, python, metadata]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/go-rust-jvm-and-cpp-reversing.md
  - ../raw/reverse/python-bytecode-esolangs-and-uefi.md
  - ../raw/reverse/UMDCTF2017-yarv-abuse-wp.md
updated: 2026-07-28
---

# Managed-Runtime Metadata and Bytecode Recovery

## 适用场景

.NET/JVM/Python/Go/Rust/Kotlin 等二进制保留类型、方法、常量池、反射或字节码元数据；恢复运行时语义和符号比从裸汇编猜算法更短。

## 识别信号

- 存在 CLR metadata、JVM constant pool、Python marshal/pyc 或 Go pclntab。
- 反编译器能恢复类型/方法，但混淆名、动态调用或版本不匹配。
- 代码由 bytecode interpreter/managed runtime 执行。

## 最小证据

- 确认运行时版本、文件/bytecode magic 和入口点。
- 导出方法表、常量池、类型关系与异常处理。
- 反编译结果可用运行时执行或字节码模拟验证。

## 解法骨架

1. 用格式专用解析器恢复 metadata 和方法边界。
2. 对照运行时版本解析 opcode、calling convention 和对象布局。
3. 重命名数据流关键方法，处理反射、动态加载和字符串解密。
4. 写最小 harness 调用目标方法，比较真实输出。

## 关键变体

- Python marshal/pyc 与版本化 opcode。
- JVM/.NET metadata、reflection 和混淆。
- Go/Rust 符号、接口/trait dispatch 与 panic/RTTI。

## 常见陷阱

- 用错误版本的 opcode 表反汇编。
- 把反编译器类型推断当作事实。
- 忽略动态加载模块和运行时生成代码。

## 关联技巧

- [go-rust-jvm-and-cpp-reversing.md](go-rust-jvm-and-cpp-reversing.md)
- [python-bytecode-esolangs-and-uefi.md](python-bytecode-esolangs-and-uefi.md)
- [cython-and-python-extension-checker-recovery.md](cython-and-python-extension-checker-recovery.md)

## 原始资料

- [go-rust-jvm-and-cpp-reversing.md](../raw/reverse/go-rust-jvm-and-cpp-reversing.md)
- [python-bytecode-esolangs-and-uefi.md](../raw/reverse/python-bytecode-esolangs-and-uefi.md)
- [UMDCTF2017-yarv-abuse-wp](../raw/reverse/UMDCTF2017-yarv-abuse-wp.md)
