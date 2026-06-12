# Elk WP

## 题目简述

题目把 Elk v3.0.0 这个嵌入式 JavaScript 引擎放进一个 `0x1000` 字节的 mmap arena 中，注册 `print` 和 `exit` 两个 C 函数，然后循环读取 JS 代码并执行。

Elk 的对象、属性和字符串都保存在 `js->mem` 这块线性内存中，`jsval_t` 使用 NaN-boxing。`T_CFUNC` 的 payload 是 48 bit C 函数指针，因此如果能伪造 JS 属性值，就有机会把 JS 函数调用转成任意 C 函数调用。

## 解题过程

漏洞点在 `setprop`。新增属性时，它先把对象的属性链头改成当前 `js->brk`，再调用 `mkentity` 分配真正的 property：

```c
jsoff_t brk = js->brk | T_OBJ;
memcpy(&js->mem[head], &brk, sizeof(brk));
return mkentity(js, (b & ~3U) | T_PROP, buf, sizeof(buf));
```

如果这时 arena 空间不足，`mkentity` 返回 OOM，但对象属性链头已经被改写为一个并不存在的实体，旧链表头只保存在局部变量里，函数返回后丢失。下一次 GC 时，从 `js->scope` 出发无法再遍历到旧属性链，GC 会把大部分实体清掉，最后只留下 scope object。此时 `js->brk` 回到很小的位置，攻击者可以通过后续字符串内容在可预测偏移布置假对象和假属性链。

利用分四个阶段：

1. 用 `print(print)` 泄露 `js_print` 地址。`print` 是 `T_CFUNC`，字符串化时会输出 `c_func_0x...`，可由此计算 PIE 基址。
2. 发送超长字符串触发 `setprop` OOM，破坏 scope 的属性链，等待下一轮 GC 清空 arena。
3. 在清空后的 arena 中用 JS 字符串的 `\xNN` 转义布置假 property 链。伪造名为 `x` 的属性，值为泄露出的 `js_print`；同时伪造 `v1`、`v2` 为越界字符串，让 `x(v1, v2)` 打印 GOT 或指针区域，泄露 libc base 和 `js_buf` base。
4. 再触发一次 OOM/GC 重置 arena，布置最终假 property：让 `x` 的值成为 libc gadget 地址。调用 `x()` 后从 `call_c` 进入 gadget 链，读取 `struct js` 中的字段，最终执行 `mov rsp, rdx; ret` 把栈迁移到 `js->mem` 中的 ROP 链。

最终 ROP 链设置：

```text
rdi = "/bin/sh"
rsi = 0
rdx = 0
rax = 59
syscall
```

这样通过 fake `T_CFUNC` 调用进入 libc gadget，栈迁移到 JS arena，完成 `execve("/bin/sh", NULL, NULL)`。

## 方法总结

本题核心是一个 OOM 后状态不回滚导致的属性链损坏。利用 GC 清理后的可预测内存布局，可以把 JS 字符串内容解释成 Elk 内部 entity，先泄露地址，再伪造 `T_CFUNC` 劫持控制流。小型解释器题要重点关注对象布局、GC 可达性、错误路径是否回滚，以及 host 函数指针如何编码。
