# SUCTF2026-minivfs

## 题目简述

题目把 16 个堆块包装成 `touch / rm / cat / write` 等 VFS 命令，使用随附件提供的 glibc 2.41。程序保护全开，并通过 seccomp 禁止 `execve`、`execveat`、`fork`、`clone`、`ptrace` 等调用，但 `open`、`read`、`write`、`getdents64` 和 `mprotect` 仍可使用。

核心漏洞有两个：

- `malloc(cap)` 得到的内存没有初始化，而 `cat` 固定输出完整 `cap`，可泄漏被复用 chunk 中残留的 libc/heap 指针；
- `write` 允许 `n == cap`，完成 `memcpy` 后仍执行 `data[n] = '\0'`，形成单字节 off-by-null，可清除相邻 chunk 的 `PREV_INUSE`。

官方利用用 off-by-null 制造堆重叠，经 largebin attack 修改 `mp_` 中的 tcache 范围，再 poison 大尺寸 tcache 到 `_IO_2_1_stdout_`，通过 FSOP、`setcontext` 和两阶段 shellcode 枚举随机 flag 文件名并读取内容。参赛队总 WP 则把 largebin 任意写直接打到 `_IO_list_all`，在退出流程触发位于堆上的 fake FILE。

## 解题过程

### 1. 还原路径槽位和认证值

每个文件路径先经过 FNV-1a 和两轮整数混合，低 4 位决定槽位，完整 hash 再与常量异或得到 token：

```python
def path_hash(path: str) -> int:
    h = 0x811c9dc5
    for byte in path.encode():
        h = ((h ^ byte) * 0x01000193) & 0xffffffff
    h ^= h >> 16
    h = (h * 0x7feb352d) & 0xffffffff
    h ^= h >> 15
    h = (h * 0x846ca68b) & 0xffffffff
    h ^= h >> 16
    return h & 0xffffffff


def slot(path: str) -> int:
    return path_hash(path) & 0xf


def token(path: str) -> int:
    return path_hash(path) ^ 0xa5a5a5a5
```

该 token 完全由公开路径决定，只是命令格式要求，不构成秘密认证。利用前应为计划使用的短路径预先计算槽位，避免两个名字映射到同一个槽而破坏堆布局。

### 2. 漏洞根因

`touch` 把请求大小限制在 `0x418..0x500`：

```c
if (cap < 0x418 || cap > 0x500)
    return -4;
g_nodes[slot].data = malloc(cap);
```

源码注释声称这里会 zero-init，但实际调用的是 `malloc` 而非 `calloc`。`cat` 又按容量而非已写长度输出：

```c
write(1, g_nodes[slot].data, g_nodes[slot].cap);
```

因此，释放一个进入 unsorted bin 的 chunk，再用相近请求取回并只覆盖前 8 字节，后续 `cat` 会把剩余的 bin 指针一并输出。这不是越过 chunk 边界的 over-read，而是未初始化内容泄漏。

真正的越界写位于 `write`：

```c
if (n > g_nodes[slot].cap)
    return -4;
memcpy(g_nodes[slot].data, input, n);
g_nodes[slot].data[n] = '\0';
```

当 `n == cap` 时，最后一个 NUL 落在下一个 chunk 的 size 低字节上。合理选择相邻请求大小后，可以把 `PREV_INUSE` 清零，并配合伪造的 `prev_size`、`size`、`fd`、`bk` 触发向后合并。

### 3. 泄漏 libc 和 heap

官方 exp 先申请四个 `0x500` 请求，释放其中两个使其进入 unsorted bin，再以 `0x420` 请求取回。新 chunk 没有清空，`cat` 会同时给出 `main_arena` 指针和堆链表指针：

```text
touch J 0x500
touch Y 0x500
touch R 0x500
touch B 0x500
rm J
rm R
touch J 0x420
touch R 0x420
cat J
```

官方 exp 对随附 libc 使用：

```python
libc.address = leaked_arena - 0x210f50
heap_base = leaked_heap - 0x290
```

参赛队题解使用了另一套堆风水，得到的示例偏移是 `arena - 0x210b00` 和 `heap_ptr - 0x1600`。这些差异来自泄漏位置和具体布局，不能把偏移当作 glibc 2.41 的通用常量；必须绑定题目提供的 `libc.so.6`，并在调试器中确认泄漏指向哪个结构字段。

### 4. off-by-null 构造 House of Einherjar

官方布局使用四个相邻请求：

```text
J: 0x4f8
Y: 0x428
R: 0x4f8
B: 0x418
```

在 J 中布置可通过 unlink 一致性检查的 fake chunk，核心字段为：

```python
base = heap_base + 0x2f0
fd = base - 0x18
bk = base - 0x10

fake  = p64(0) * 5
fake += p64(0x901)
fake += p64(fd) + p64(bk)
fake += p64(0) * 2
fake += p64(heap_base + 0x2c0)
```

随后：

1. 释放 J，使前方 fake chunk 对应区域成为可合并目标；
2. 对 Y 写满 `0x428` 字节，末尾 NUL 清掉 R 的 `PREV_INUSE`；
3. 同时把 R 的 `prev_size` 修成伪造范围；
4. 释放 R，触发向后合并和 unlink；
5. 重新申请不同大小的 chunk，使两个 VFS 节点获得重叠区域。

从此可以通过仍在使用的节点修改另一个空闲 chunk 的 `fd/bk` 或 `bk_nextsize`，为 largebin attack 准备任意地址写。

### 5. 官方路线：扩大 tcache 后毒化 `stdout`

官方 exp 选择 `libc + 0x2101e8` 附近的 `mp_` tcache 控制字段作为 largebin 写目标。先把较大 chunk 排入 largebin，再篡改重叠 chunk 的 `bk_nextsize = target - 0x20`；下一次把 unsorted chunk 插入 largebin 时，glibc 会把 chunk 地址写到目标位置。

这次写入扩大了可进入 tcache 的尺寸范围，使 `0x4c0` 请求也能进入对应 tcache bin。之后：

```text
申请两个 0x4c0 chunk
依次释放进入 tcache
通过 overlap 改写首个空闲 chunk 的 next
next = _IO_2_1_stdout_ ^ (chunk_addr >> 12)
连续申请两次，第二次返回 stdout
```

最后一行是 glibc safe-linking 编码；若 chunk 与 heap base 不在同一页，应使用实际 `chunk_addr >> 12`，不能机械使用 `heap_base >> 12`。

在 `_IO_2_1_stdout_` 上伪造 wide FILE，令 `_IO_wfile_jumps` 路径间接调用 `setcontext+61`。官方 exp 恢复一个执行 `mprotect(heap, 0x10000, 7)` 的上下文，把后续 shellcode 所在堆页改成可执行，再进入两阶段 shellcode：

1. `open("/") → getdents64 → write`，输出根目录并找到随机名称 `flag_<hex>`；
2. 从 stdin 接收第二段 shellcode，执行 `open(flag_path) → read → write`。

程序后续调用 `printf` 时触发被改写的 `stdout`，所以无需等待进程退出。

### 6. 参赛队路线：largebin 写 `_IO_list_all`

参赛队总 WP 使用相同的泄漏与 off-by-null overlap，但将被篡改的 `bk_nextsize` 指向：

```text
_IO_list_all - 0x20
```

下一轮 largebin 插入把堆上 fake FILE 的地址写入 `_IO_list_all`，发送 `quit` 后在 libc 清理 FILE 链时触发。该方案的关键布置为：

```text
fake FILE:
  +0x78  setcontext frame / 后续 ROP 位置
  +0x88  可写 lock
  +0xa0  fake wide_data
  +0xa8  pop rdx ; leave ; ret
  +0xc0  _mode = 1
  +0xd8  _IO_wfile_jumps
  +0xe0  fenv 指针
  +0x1c0 mxcsr = 0x1f80

fake wide_data:
  +0xe0  自定义 wide vtable
  vtable+0x68 = setcontext
```

这里使用完整 `setcontext`，再以 `pop rdx ; leave ; ret` 完成第二次栈迁移。原因是该调用路径中的 `rdx` 不稳定，直接跳常见的 `setcontext+0x3d` 会让上下文恢复从错误地址取值；同时 `fenv` 和 `mxcsr` 必须有效，否则恢复浮点状态时会崩溃。

这条路线用 ROP 直接执行 syscall，不需要先 `mprotect`。第一次连接以 `open(".", O_DIRECTORY) → getdents64 → write` 枚举文件名，第二次重建同样的堆布局，再执行 `open → read → write` 读取目标文件。

### 7. flag 结果

远程目录中的真实文件名带随机后缀，不能照搬本地的 `/flag`。参赛队样例先枚举到：

```text
./flag_78f16013a3c04854
```

随后读取得到：

```text
flag{min1_vfs_5afe_b4ck3nd_chunk5_h1dd3n_s3cre7_SUCTF_2026}
```

## 方法总结

本题的入口是“未初始化整块输出 + 单字节 NUL 越界”：前者给出 libc/heap，后者清除 `PREV_INUSE` 并通过 House of Einherjar 制造 overlap。获得重叠后，官方路线利用 largebin 写 `mp_`，扩大 tcache 范围并 poison 到 `stdout`；另一条路线则直接 largebin 写 `_IO_list_all`，在退出流程触发堆上的 fake FILE。

复现时最容易出错的是把具体偏移和堆地址关系当成固定公式。glibc build、申请序列、bin 状态和泄漏字段任何一个变化，`main_arena`、heap 指针、safe-linking 编码和 fake FILE 调用路径都会变化。应始终使用附件 libc，在每个阶段核对 chunk header、bin 链、目标指针和触发点。由于 flag 文件名随机，最终链还必须包含目录枚举，而不是只写死 `/flag`。
