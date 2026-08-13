# The Motorala 2

## 题目简述

源码与前一题相同，但被编译为 WASI WebAssembly 并由 Wasmtime 运行。无界 `scanf("%s", attempt)` 不再覆盖原生返回地址，而是在 WASM 线性内存中从栈缓冲区一路写向 dlmalloc 元数据和保存 PIN 的堆缓冲区。目标是保持中间元数据不变，同时把 `attempt` 与 `pin` 都变成空字符串。

## 解题过程

调试题目给定的 Wasmtime 19.0.1 环境可确认：`attempt` 起始于线性内存对应宿主地址 `0x7ffe700122d0`，PIN 缓冲区起始于 `0x7ffe700127e0`。两者相距 1296 字节，而且 WASM 线性内存没有进程级 ASLR，布局在相同环境中可重复。

不能简单发送 1296 字节填充，因为中途覆盖 dlmalloc 的关键结构会先让程序崩溃。官方解法在调试器中保存两点之间除第一个字节外的原始内容：

```text
dump memory out.bin 0x7ffe700122d1 0x7ffe700127e0
```

得到的 `out.bin` 正好是 1295 字节。构造输入时，先放一个 NUL 让 `attempt` 立即成为空 C 字符串，再原样回放中间内存。dump 中若含换行字节，需要换成 NUL，避免 `%s` 把它当作输入终止符：

```python
saved = open("out.bin", "rb").read().replace(b"\n", b"\x00")
payload = b"\x00" + saved
io.sendlineafter(b"PIN: ", payload)
```

`scanf` 写完 1296 个输入字节后，还会在下一地址追加字符串终止 NUL；下一地址恰好是 PIN 缓冲区开头。于是 `attempt` 和 `pin` 都以 NUL 开头，`strcmp(attempt, pin) == 0`，而中间运行时元数据被完整还原。程序进入 `view_message()` 并输出：

```text
grey{s1mpl3_buff3r_0v3rfl0w_w4snt_1t?_r3m3mb3r_t0_r34d_th3_st0ryl1ne:)}
```

## 方法总结

WASM 提供控制流完整性，使经典覆盖原生返回地址的打法失效，但 C 代码的线性内存越界仍能破坏同一地址空间内的其他对象。这里利用的不是控制流劫持，而是跨越栈与堆篡改认证数据；固定布局又允许把途中必须保留的 allocator 元数据逐字节回放。复现必须匹配题目指定的 Wasmtime 和构建产物。
