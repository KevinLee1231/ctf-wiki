---
type: technique
tags: [pwn, kernel, uaf, race, slab, object-lifetime]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/kernel-uaf-race-and-slab-techniques.md
  - ../raw/pwn/linux-kernel-exploit-basics.md
  - ../raw/pwn/D3CTF2019-knote-v1-v2-wp.md
updated: 2026-07-28
---

# Kernel Object Lifetime and Race Window

## 适用场景

内核对象在并发、引用计数、RCU、ioctl 或异步回调中提前释放/重复使用，可通过堆喷、阻塞原语和跨 cache 复用形成 UAF、double fetch 或对象类型混淆。

## 识别信号

- create/use/delete 在多线程下缺少锁或引用持有。
- 用户拷贝、页错误、FUSE/userfaultfd 可扩大检查到使用窗口。
- freed slab 对象可被 msg/pipe/keyring/同 cache 对象占用。

## 最小证据

- 给出对象状态机、引用变化和具体竞态交错。
- 确认目标 kernel/slab 配置、cache 大小与可用喷射对象。
- 在无提权副作用条件下稳定复现 UAF/重用。

## 解法骨架

1. 用日志/断点确定释放、回调和使用时序。
2. 选择可控阻塞点扩大窗口，并用线程同步固定交错。
3. 释放后立即喷射兼容对象，验证字段重解释。
4. 将 primitive 转为泄露、任意读写或 page overlap，再处理保护和提权目标。

## 关键变体

- Refcount/RCU UAF。
- Double fetch / TOCTOU。
- Slab cross-cache、pipe/page/PTE overlap。

## 常见陷阱

- 仅靠 `sleep` 碰竞态，成功率不可复验。
- 喷射对象不在同一 cache 或被隔离策略阻断。
- 忽略 SMEP/SMAP/KASLR/KPTI 和内核版本差异。

## 关联技巧

- [kernel-uaf-race-and-slab-techniques.md](kernel-uaf-race-and-slab-techniques.md)
- [linux-kernel-exploit-basics.md](linux-kernel-exploit-basics.md)
- [race-condition-and-concurrency-exploits.md](race-condition-and-concurrency-exploits.md)
- [kaslr-kpti-smep-and-kernel-debugging.md](kaslr-kpti-smep-and-kernel-debugging.md)

## 原始资料

- [kernel-uaf-race-and-slab-techniques.md](../raw/pwn/kernel-uaf-race-and-slab-techniques.md)
- [linux-kernel-exploit-basics.md](../raw/pwn/linux-kernel-exploit-basics.md)
- [D3CTF2019-knote-v1-v2-wp](../raw/pwn/D3CTF2019-knote-v1-v2-wp.md)
