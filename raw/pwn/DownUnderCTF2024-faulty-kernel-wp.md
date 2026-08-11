# Faulty Kernel

## 题目简述

内核模块 `/dev/challenge` 为每次打开分配 128 个 page 指针，并允许用户以 `mmap` 映射这些页。VMA fault handler 的上界判断写成 `if (pgoff > sbuf->pagecount)`；当 `pgoff == 128` 时错误地放行，随后访问 `sbuf->pages[128]`，即指针数组后一项的越界内核读。通过 slab/pipe grooming，可让该位置被解释成攻击者关联的 `pipe_buffer` page，并把它映射回用户态获得对文件页的写入能力。

内核启动参数含 SMEP、SMAP、KASLR 与 KPTI，但官方解法无需劫持内核控制流：目标是覆写 `/etc/passwd` 后通过普通 `su root` 获得 flag，因此归入 Pwn。

## 解题过程

### 形成越界页映射

先以 128 个页建立合法映射，再对该映射作 `mremap(..., MREMAP_MAYMOVE)` 扩大两倍。原 VMA 保留 challenge 的 fault handler，访问新增的第 129 页时 `pgoff` 为 128；错误的 `>` 而非 `>=` 检查允许索引越界。

官方 exploit 绑定单 CPU，并用大量 pipe 填充 `kmalloc-1024`，交替缩小 pipe 创建洞。它将只读映射的 `/etc/passwd` 页通过 `vmsplice` 放进另一批 pipe 的 `pipe_buffer`，随后打开 challenge 并映射。这样 `sbuf->pages[128]` 的邻接堆对象有机会提供该 pipe 的 page 指针。

### 以文件页篡改完成提权

`mremap` 后的第二半映射被当作 `passwd_str`；exploit 先检查前四字节是否为 `root`，以避免邻接对象没有按预期命中时盲写。命中后，将开头替换为：

```c
"root::0:0:xroot:/root:/bin/sh\0"
```

这删除 root 账户的密码字段。官方交付脚本把静态编译 exploit 上传到 QEMU guest，运行后执行 `su root` 再读取 flag。

### 验证

题目配置给出 `DUCTF{0u7-f4ul71n6_7h3_f4ul7_h4ndl3r}`。本文没有启动 QEMU、加载内核模块或执行 exploit；页面相邻布局与成功条件均来自模块源码及官方 exploit 的静态对照。

## 方法总结

- 核心技巧：VMA 映射长度与 fault handler 中的逻辑 pagecount 必须共同检查；一位边界错误即可把内核堆邻接指针变成用户页映射。
- 识别信号：自定义 `mmap` 驱动出现 `pgoff > count`、可 mremap 的 mapping 与 `struct page **` 数组时，应优先检查 off-by-one fault。
- 复用要点：内核堆布局和 pipe 结构严重依赖内核版本；先用可验证的 `memcmp("root")` 门槛确认 page 命中，比对未知邻接页直接写入更安全。
