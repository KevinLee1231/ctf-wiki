---
type: family
tags: [pwn, family, kernel, uaf, race, slab, ebpf]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/kernel-uaf-race-and-slab-techniques.md
  - ../raw/pwn/WMCTF2025-wm-easynetlink-wp.md
updated: 2026-06-12
---

# Kernel UAF, Race and SLAB Techniques

## 作用边界

本页是 Linux kernel UAF、race、SLUB/slab、对象复用和 eBPF 扩 primitive 的二级 family。它从 [linux-kernel-exploit-basics.md](linux-kernel-exploit-basics.md) 进入，负责把“内核漏洞”进一步分成对象生命周期、竞态窗口、cache 复用、freelist hardening、页表/PTE 和 BPF verifier/runtime 差异路线。

## 识别信号

- 漏洞触发面是 ioctl、netlink、procfs、VFS/FUSE、eBPF、字符设备或内核模块对象。
- 已经有 UAF、double free、随机写、race window、slab overlap、page overlap 或 BPF map value 异常。
- 目标对象涉及 `tty_struct`、`seq_operations`、`msg_msg`、`pipe_buffer`、`bpf_array`、PTE、file 结构或驱动私有对象。
- 保护中出现 SLUB hardening、freelist random/obfuscation、KASLR、KPTI、SMEP/SMAP 或 `userfaultfd` 限制。

## 最小证据

- 漏洞对象的 cache size、分配/释放路径和可控字段。
- 可喷对象列表以及每个对象能提供的能力：leak、AAW、function pointer、file ops、map value、pipe page。
- 竞态题要有命中率、窗口位置和稳定化手段；不能只靠偶发成功。
- 提权目标已选定：kernel ROP、cred overwrite、`modprobe_path`、PTE remap、BPF AAR/AAW 或 file write。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| `tty_struct` / fake vtable | 对象 cache 是否可重占，函数指针是否可控 | kROP 或 fake ops |
| ioctl register / AAW | 写入粒度、地址对齐和写前 leak 是否足够 | `modprobe_path`、cred、PTE 或 ROP 栈 |
| `userfaultfd` race | 远程内核是否允许 uffd，窗口是否可暂停 | uffd 失败时转 mprotect/MADV/线程调度 |
| SLUB freelist hardening | freelist 编码、random 和 object size 是否已知 | 先 leak key/base，再做 overlap |
| random UAF write | 写入大小/偏移不可控但可大量喷射 | 找可容忍随机污染的 BPF/map/pipe 对象 |
| BPF verifier/runtime desync | verifier 认为安全但 runtime 值可越界 | map OOB -> AAR/AAW -> kernel write |
| PTE/page overlap | 可控页表项、物理页或 pipe page 复用 | 改页权限或映射敏感物理页 |
| panic/oops leak | crash 输出可泄露地址但可能终止服务 | 先看 `oops=panic` 和重启策略 |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2025-wm-easynetlink-wp](../raw/pwn/WMCTF2025-wm-easynetlink-wp.md) | UAF 后只能向随机释放块随机偏移写随机值时，`bpf_array` 是可控大小的占位对象；污染 map value 后可借 BPF verifier 认知差异转成 AAR/AAW。 |
| [D3CTF2019-knote-v1-v2-wp](../raw/pwn/D3CTF2019-knote-v1-v2-wp.md) | 内核 note 对象 UAF/堆布局是主线，先确认 slab 对象复用和提权目标。 |
| [D3CTF2022-d3kheap-wp](../raw/pwn/D3CTF2022-d3kheap-wp.md) | 内核堆 UAF 与 msg_msg/pipe_buffer 等对象复用，先固定 slab 布局和提权写点。 |
| [D3CTF2023-d3kcache-wp](../raw/pwn/D3CTF2023-d3kcache-wp.md) | 页级 UAF、buddy/slab/pipe_buffer 复用是主线，先把 page 到 pipe 的重分配稳定化。 |
| [D3CTF2025-d3kheap2-wp](../raw/pwn/D3CTF2025-d3kheap2-wp.md) | 新版内核堆题继续围绕 UAF/page/slab 复用，先确认对象释放窗口和可控映射。 |
| [D3CTF2025-d3kshrm-d3kshrm-revenge-wp](../raw/pwn/D3CTF2025-d3kshrm-d3kshrm-revenge-wp.md) | 共享内存 mmap fault 下标越界可映射相邻 struct page，先稳定伪造页表和内核任意映射。 |

## 合并与拆分结论

- 保留为 family：本页连接 UAF、race、SLUB、BPF、PTE 和对象喷射多条 kernel 路线。
- 不合并进 `linux-kernel-exploit-basics.md`：kernel 总入口负责环境和保护首轮，本页负责生命周期 primitive 的二级分流。
- 暂不拆 eBPF 或 pipe/PTE 小页：当前 raw 多作为 kernel UAF/race 的落地路线存在，还不足以独立成 technique。

## 常见误判

- 只知道有 UAF，不确认 cache size 和可喷对象，导致远程布局不可控。
- 把 `userfaultfd` 当默认稳定器，没检查远程内核是否禁用。
- Freelist hardening 下直接写 fd 指针，忘记编码和 random。
- BPF 路线只看 map OOB，没有解释 verifier 和 runtime 状态为什么分叉。

## 关联页面

- [linux-kernel-exploit-basics.md](linux-kernel-exploit-basics.md)
- [kaslr-kpti-smep-and-kernel-debugging.md](kaslr-kpti-smep-and-kernel-debugging.md)
- [cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md)
- [race-condition-and-concurrency-exploits.md](race-condition-and-concurrency-exploits.md)
- [pwn-tooling.md](pwn-tooling.md)

## 原始资料

- [kernel-uaf-race-and-slab-techniques.md](../raw/pwn/kernel-uaf-race-and-slab-techniques.md)
- [WMCTF2025-wm-easynetlink-wp](../raw/pwn/WMCTF2025-wm-easynetlink-wp.md)
