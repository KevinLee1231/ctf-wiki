# Pwn The Toilet

## 题目简述

程序最多分配 5 个 `0x800` 内容块，每个内容块另有一个 16 字节 `Poop` wrapper。释放时只释放内容、不清空 wrapper 或内容指针；修改功能的偏移检查又误用了 `idx`，使任意非负 offset 都可通过，从而获得堆上越界写。

## 解题过程

先申请 5 组大块，释放最后一组。`view_poop` 仍会打印已释放内容地址和字符串；`0x800` 块进入 unsorted bin，内容中的 arena 指针同时给出 libc 泄漏，wrapper 打印的原地址给出 heap 泄漏。

第一次 `mess_with_poop` 选择块 0，使用接近块尾的 offset，让最多 `0x800` 字节的 `read` 越过内容边界，覆写相邻 wrapper 的 `poopsize` 与 `poopAddress`。把 `poopAddress` 改到 glibc 的退出处理结构，可通过 `view_poop` 读取编码后的 `__exit_funcs` 函数指针。

glibc 指针保护形式为：

$$
encoded=\operatorname{rol}_{64}(pointer\oplus guard,17).
$$

已知编码值和原函数地址，就能反推出 `guard`：

$$
guard=\operatorname{ror}_{64}(encoded,17)\oplus pointer.
$$

第二次也是最后一次修改，把退出函数指针编码为 `system`，并把参数槽改成 libc 中的 `/bin/sh`。选择菜单的 Leave 触发 `exit`，得到 shell并读取：

```text
grey{1_L0v3_p0oP1Ng_4nD_s0_Do_y0u!_90531y057gioab}
```

## 方法总结

利用链同时用到了 UAF 泄漏、错误变量导致的无界 offset、堆元数据覆盖和 glibc exit handler 指针加密。由于修改次数限制为 2，第一次必须建立任意地址读取并取得 pointer guard，第二次一次性完成目标覆盖。修复应在 free 后清空内容指针／wrapper，并把检查改为 `offset < curpoopsize`，同时限制写入长度为剩余空间。
