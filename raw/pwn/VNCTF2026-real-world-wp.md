# Real world?

## 题目简述

题目暴露的是未修改的 Rsync 3.3.0 服务，属于真实漏洞复现型 pwn。附件和服务行为都指向标准 rsync 协议实现，主要利用链来自两个公开漏洞：`CVE-2024-12085` 未初始化数据泄漏和 `CVE-2024-12084` checksum 解析堆溢出。

本题目标不是重新挖洞，而是把公开 advisory 中的 primitive 映射到比赛环境：先用 `CVE-2024-12085` 泄漏栈、堆和程序地址，绕过 ASLR/PIE；再用 `CVE-2024-12084` 构造任意地址写。最终把写原语落到栈上，布置控制流或 ROP 达到 RCE。

## 解题过程

### 源码

Rsync 3.3.0
[https://github.com/RsyncProject/rsync](https://github.com/RsyncProject/rsync)

### 分析

这题和强网杯2025决赛rsync 一样，题目是没有任何变化的rsync 3.3.0，漏洞为
CVE-2024-12084（堆溢出）
CVE-2024-12085（未初始化数据泄漏）
[https://github.com/google/security-research/security/advisories/GHSA-p5pg-x43v-mvqj](https://github.com/google/security-research/security/advisories/GHSA-p5pg-x43v-mvqj)

`CVE-2024-12084` 出在 checksum 解析流程。协议协商后的 `s2length` 可控，而结构体中的 `sum2` 固定为 16 字节；在支持 SHA256/SHA512 等 digest 的构建中，`MAX_DIGEST_LEN` 可能大于 16，于是服务端读入 checksum 时可向 `sum2` 后越界写入，形成堆溢出。

`CVE-2024-12085` 出在本地计算 checksum 后的比较流程。栈上的 `sum2` 缓冲区没有完全初始化，攻击者可以控制比较长度，让服务端把已知 checksum 前缀与未初始化栈数据逐字节比较，从而逐步泄漏栈、堆、程序地址等 ASLR 相关信息。

利用时先通过 `CVE-2024-12085` 泄漏栈、堆和程序地址；再用 `CVE-2024-12084` 把堆溢出转成任意地址写。最终利用任意地址写覆盖栈上控制数据，布置 ROP 或等价控制流达到 RCE。

## 方法总结

真实软件题要先确认版本是否“原样未改”，再把公开 advisory 中的 primitive 映射到比赛环境。这里两个漏洞正好形成完整链路：信息泄漏负责定位，堆溢出负责写入，栈上落点负责劫持执行流。写 WP 时保留 CVE 编号和 primitive 比保留一次性靶机细节更有复用价值。
