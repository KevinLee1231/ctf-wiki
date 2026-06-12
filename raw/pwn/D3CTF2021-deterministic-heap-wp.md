# Deterministic Heap

## 题目简述

本题是 Windows 平台堆利用题，围绕 NT Heap 的低碎片堆（LFH）稳定占位展开。题目本身存在 UAF/重叠对象利用条件；如果不追求成功率，可以反复尝试碰撞占位，但预期解要求通过 LFH bucket、subsegment 和 `CachedItems` 的结构规律稳定把释放块重新申请回来。

题目附件包含 Windows 堆利用源码，其中核心辅助逻辑围绕 LFH bucket 与可预测申请/释放模式展开。核心目标是在最大尺寸 LFH bucket 中制造可预测占位，得到重叠对象后泄露 vtable、程序基址、系统库地址和栈返回地址，最后把返回地址改成调用 `system("cmd.exe")` 的 ROP。

## 解题过程

题目源码和 exp 见 [`ArisXu/Deterministic-Heap`](https://github.com/ArisXu/Deterministic-Heap)。下文保留了仓库中最关键的 LFH 稳定占位方法、版本相关偏移和最终 ROP 参数。

本题是一次在windows平台利用一个堆溢出漏洞最后编写稳定exp时被低碎片堆(LFH)难住的问题,这题的设计也是在简化的同时尽量还原了当时的情况。稳定的低碎片堆利用在过去的win10版本中曾有人研究过 [Deterministic_LFH](https://github.com/saaramar/Deterministic_LFH),这个方法非常有趣但很遗憾在现在的nt heap上已经失效了。

如果不追求高成功率的漏洞利用,本题只是一道简单的uaf利用,哪怕不进行任何的堆布局也会有一定的概率成功进行堆块的占位,但考虑到一次攻击可能需要较多的交互次数(这是设计上的失败),所以我仅将需要的连续攻击次数设置成了3次,而且由于时间有点赶没来得及把挑战次数限制设置上,所以其实是可以以较低概率的利用脚本重复尝试直到运气好成功。

预期解的原理不复杂，但有明确限制：它主要适用于本题所用的最大尺寸 LFH bucket，不一定能直接迁移到其他尺寸。

下面把预期解的堆布局逻辑写成可复用步骤；实际复现时仍建议配合调试观察 LFH bucket、subsegment 和 `CachedItems` 的变化。

在清楚 LFH 结构的基础上（可以结合 [Windows 10 NT Heap Exploitation](https://www.slideshare.net/AngelBoy1/windows-10-nt-heap-exploitation-chinese-version) 理解），最大尺寸 LFH bucket 内的 chunk 数量最大为 `0x10` 个，即 `0x10 * 0x4000 = 0x40000`；segment 的 `CachedItems` 数量也是 `0x10` 个。通过连续申请 `0x10` 个 subsegment 数量的 chunk，即 `0x10 * 0x10 = 0x100` 个 chunk，再间隔 1 个 subsegment 释放 chunk，可以保证每个释放的 chunk 来自不同 subsegment，并且这些 subsegment 都只有一个 free chunk。所有这些 subsegment 都会进入 `CachedItems`，接着申请 `0xf` 个 chunk 后，下一个 chunk 就一定来自上述 subsegment。于是下图的稳定占位关系可以成立；图中虽然是 OOB 场景，UAF 场景里红色和黑色区域对应同一块释放后重占的内存。

![LFH 稳定占位布局示意](<./D3CTF2021-deterministic-heap-wp/lfh_bucket_layout.png>)

这是在不能重复触发漏洞但堆块大小不受限制时较稳定的占位方法，测试中基本可以做到 100% 成功。后续利用不需要长期 WP 保留整份远程 exp，只需要保留关键 primitive：

1. 通过 LFH 布局使释放块与新对象重叠，找到 magic 不一致的对象索引，确认 overlap 成功。
2. 利用重叠的 integer/string 对象泄露 vtable，计算程序基址。
3. 构造任意读写：把重叠对象中的指针改到 `addr - 4`，用 show/edit 读写目标地址。
4. 根据 Windows 版本差异，从 IAT/PEB/TEB 泄露 `kernel32`、`ucrtbase`、`ntdll` 和栈地址。
5. 在栈上搜索目标返回地址，把返回地址改成调用 `system("cmd.exe")` 的 ROP，再读取 flag。

PDF exp 中和环境强相关的偏移也应保留。泄露 vtable 后先算程序基址：

```python
cbase = vtable - 0x55dc
```

再按 Windows 版本选择系统库和栈地址恢复方式：

```text
20H2:
  kernel32 = read(cbase + 0x5024) - 0x19910
  ucrtbase = read(cbase + 0x50f4) - 0x31980
  ntdll    = read(kernel32 + 0x81bf0) - 0x4cc30
  peb      = read(ntdll + 0x125d34) - 0x44

2004:
  kernel32 = read(cbase + 0x5024) - 0x19910
  ucrtbase = read(cbase + 0x50f4) - 0x31980
  ntdll    = read(kernel32 + 0x81bf0) - 0x40cc0
  peb      = read(ntdll + 0x125d34) - 0x44

1909:
  kernel32 = read(cbase + 0x5024) - 0x1f4d0
  ucrtbase = read(cbase + 0x50e8) - 0x2edb0
  ntdll    = read(kernel32 + 0x81b54) - 0x3fcb0
  peb      = read(ntdll + 0x11dc54) - 0x44
```

如果 `peb` 低 16 bit 读成 0，PDF exp 的做法是再读高 16 bit 后左移拼回：

```python
if peb == 0:
    peb = read(peb_symbol + 2) << 16
```

随后由 `TEB = PEB + 0x3000` 读栈指针，在栈上搜索返回地址标记 `cbase + 0x18be`。找到返回地址后写入调用 `system("cmd.exe")` 的 ROP：

```python
system = ucrtbase + 0xed060
rop = p32(system) + p32(0xdeadbeef) + p32(ret + 0x10) + p32(0)
rop += b"cmd.exe\x00"
write(ret, rop)
```

## 方法总结

- 核心技巧：利用 LFH 最大 bucket 中 subsegment 和 `CachedItems` 的可预测行为稳定占位，把 UAF 转成对象重叠，再构造任意读写和 ROP。
- 识别信号：Windows 堆题要求多次稳定利用，目标块大小落在 LFH bucket，释放后需要精确把同尺寸块重新申请回来。
- 复用要点：LFH 利用的关键不是单次申请/释放，而是控制同 bucket 中多个 subsegment 的空闲块分布；长期 WP 中不要保留比赛远程 IP，复现时用本地服务或占位 host/port 替换。

