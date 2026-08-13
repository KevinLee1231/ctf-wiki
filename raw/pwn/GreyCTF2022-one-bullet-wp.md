# GreyCTF2022 - One Bullet

## 题目简述

程序直接泄露 `system` 地址，并提供一次任意 8 字节读取以及一次有限长度栈覆盖。二进制启用栈 canary，服务又用 seccomp 禁止直接 `execve`；需要先读出 canary，再用分阶段 ROP 扩大输入并执行 open-read-write。

## 解题过程

根据泄露的 `system` 求 libc 基址。任意读原语对准当前栈帧 canary 槽，取得保护值。首段溢出空间只有 `0x28` 字节左右，只布置保留 canary、恢复栈并再次调用 `read` 的小型 ROP，使第二阶段可以写入更大的可控区域。

```python
libc.address = leaked_system - libc.sym['system']
canary = arbitrary_read(canary_addr)

stage1 = flat(
    b'A' * offset,
    canary,
    saved_rbp,
    rop_read_into_bss,
    pivot_to_bss,
)
```

第二阶段在可写区放置 `flag.txt\x00`，并组合 `open`、`read`、`write` 的调用链；文件描述符按服务环境的返回值传递。由于 seccomp 允许文件 I/O，该链可以直接回显：

```text
grey{0ne_5ho1_1_bo0m_d34a1}
```

## 方法总结

受限长度溢出适合“先引导、后展开”：第一阶段只做再次读取和栈迁移，复杂逻辑放到第二阶段。存在 seccomp 时应先确认允许的系统调用，不能默认拿到 libc 地址就使用 `system` 或 `execve`。
