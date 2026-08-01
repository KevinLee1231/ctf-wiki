# GlacierCTF2023 - 35 Shades of Wasm

## 题目简述

宿主使用 Wasmtime 6.0.0、Cranelift 0.93.0，并强制静态 Wasm 线性内存。服务接收 Base64 编码的 Wasm 模块并直接执行。该版本受 CVE-2023-26489 影响：Cranelift 对特定 `i32` 移位地址表达式生成代码时没有正确截断，导致边界检查和实际访存地址不一致。

## 解题过程

漏洞触发形式可写成：

```wat
(i32.load (i32.shl (local.get 0) (i32.const 3)))
```

若用 C 生成 payload，必须阻止编译器把移位结果拆到临时值。官方利用用 `noinline` 封装 `addr >> 2`、再 `<< 2` 的访问；移两位仍保留 4 字节粒度，又能让未截断的高位越过 Wasm 线性内存：

```c
__attribute__((noinline)) size_t oob_read_shift2(int64_t addr) {
    size_t base = addr >> 2;
    return *(size_t *)(base << 2);
}

__attribute__((noinline)) void oob_write_shift2(int64_t addr, size_t value) {
    size_t base = addr >> 2;
    *(size_t *)(base << 2) = value;
}
```

利用先在 Wasm 内存下方的 OOB 区域读取宿主指针，计算线性内存基址；再从固定相对位置取得动态链接器指针，按页向低地址搜索 ELF magic，定位 `ld` 和 libc。取得 libc 基址后得到 `system`，并在可写区放置函数指针。

最后伪造动态链接器的可写状态，使进程退出时 `_dl_call_fini` 的间接调用落到 `system`。相关寄存器布局让 `rdi` 指向伪造状态开头，因此写入字符串 `" /bin/sh"`；前导空格既帮助满足指针算式，也能被 shell 正常解析。模块返回触发 fini 路径后获得 shell 并读取：

```text
gctf{wasm't_so_hard_or_was_it?}
```

[GitHub 安全公告](https://github.com/advisories/GHSA-ff4p-7xrq-q5r8)保留了受影响版本、移位截断错误和越界读写影响；本题所需的触发、地址泄漏与 fini 劫持链已在正文完整说明。

## 方法总结

Wasm 沙箱安全依赖 JIT 对类型宽度和边界检查的精确实现。拿到 OOB 后仍需把相对地址原语升级为宿主模块定位、任意读写和可触发的控制流劫持；这里选择退出阶段的 `_dl_call_fini`，避免直接修改只读 GOT。
