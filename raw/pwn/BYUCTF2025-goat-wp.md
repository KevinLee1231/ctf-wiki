# Goat

## 题目简述

64 位 ELF 关闭 PIE，启用 NX、栈保护和 Partial RELRO。程序把用户第一次输入经 `snprintf` 写入 `intro`，随后直接执行 `printf(intro)`，形成一次格式化字符串漏洞。之后还会再次调用 `snprintf` 处理确认输入。

漏洞只能使用一次：泄露地址后没有机会再提交利用格式串，因此需要直接修改非 PIE 地址；Partial RELRO 使已解析的 GOT 表仍可写。

## 解题过程

漏洞发生前 `snprintf` 已被调用，其 GOT 项已经完成动态解析。目标 libc 中 `snprintf` 与 `system` 的高位通常相同，只需把 GOT 地址的低 2 字节覆盖为 `system` 的已知低位 `0x1740`。ASLR 会影响更高的半字节，因此这是约 $1/16$ 到 $1/32$ 成功率的概率利用。

格式串参数偏移为 8，`intro` 在用户数据前已经输出 24 字节，官方 payload 为：

```python
payload = fmtstr_payload(
    8,
    {elf.got["snprintf"]: p16(0x1740)},
    numbwritten=24,
    write_size="short",
)
sendline(payload)
```

下一轮把确认内容设置为 `/bin/sh\x00`。进入否定分支之外的正常结束路径时，程序原本执行：

```c
snprintf(buf, 0x3f, "\nSorry, you're not the %s...", name);
```

GOT 被篡改后，这次调用实际变为 `system(buf)`。由于第一个参数仍指向刚输入的确认缓冲区，程序执行 `system("/bin/sh")` 并获得 shell。远端前置 PoW 只需执行服务给出的计算命令；若连接因错误 ASLR 猜测而结束，则重新连接。

读取 flag 得到：

```text
byuctf{n0w_y0u're_the_g0at!}
```

## 方法总结

- 核心技巧：在只有一次格式串输入的条件下，对已解析且可写的 GOT 项做 libc 函数低 2 字节概率覆盖。
- 识别信号：No PIE、Partial RELRO、目标函数已 lazy-bind，且目标 libc 中原函数与 `system` 地址接近时，应考虑 partial GOT overwrite。
- 复用要点：低字节常量与成功概率都依赖题目 libc；需确认格式串偏移、已有输出字节数及被替换函数后续参数是否能形成有效命令。
