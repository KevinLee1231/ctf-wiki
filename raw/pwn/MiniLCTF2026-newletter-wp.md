# newletter

## 题目简述

程序维护十封“信件”，每项由堆指针和记录大小组成，提供 add/delete/edit/show/sort；菜单中还隐藏了需要 `auth == "rf"` 才能使用的 move。漏洞链由三个部分组成：`sort_chunk` 把第 11 个伪元素当成合法记录，越界交换到相邻全局变量；`realloc(NULL, 0)` 产生记录大小为 0 的最小堆块；`move` 在复制空字符串失败时提前返回，留下两个指向同一 chunk 的别名。

目标为 amd64 PIE ELF，保护为 Full RELRO、栈 canary、NX 和 PIE 全开，运行环境基于 Ubuntu 24.04。作者题解说明可以用 tcache 等多种方式放大 UAF，测试路线使用 House of Some；仓库没有保存 exp、libc 或实际 flag，因此本文只把源码能证明的 primitive 与作者明确给出的最终技术路线写实，不伪造版本相关偏移。

## 解题过程

### 利用排序越界把 `auth` 改成 `rf`

全局变量在 PIE 映像中的相对偏移为：

```text
chunk[10]  0x4060 - 0x40ff，共 10 × 0x10 字节
last_size  0x4100
auth       0x4108
```

排序内层循环写成 `j < 10 - i`。当 `i=0, j=9` 时会访问 `chunk[10]`：它的 `ptr` 实际别名 `last_size`，`size` 实际别名 `auth` 的八字节内容。初始化后的 `auth="user"` 按小端整数解释为很大的非零值，因此这个伪元素会参与交换。

预先添加一封大小为 `0x6672` 的信。排序会先把它一路移动到 `chunk[9]`，再与伪造的 `chunk[10]` 交换。真实记录进入第 11 个位置后，其 `size=0x6672` 被写到 `auth`：

```text
0x6672（小端） → 72 66 00 00 ... → "rf\0"
```

这会同时把该记录的堆指针写进 `last_size`，导致后续 add 的递增大小检查基本无法满足。因此，后续利用需要的堆块应在触发 sort 前全部布置好，不能在认证绕过后才临时补分配。

### 构造 size 为 0 的隐藏 chunk

对从未使用的槽位调用 delete 时，`chunk[idx].ptr` 为 NULL，于是：

```c
chunk[idx].ptr = realloc(NULL, 0);
chunk[idx].size = 0;
```

在题目使用的 glibc 行为下，这等价于一次最小尺寸分配，返回非 NULL 指针，但程序把逻辑大小记为 0。这个“存在指针、大小为 0”的对象能绕过 delete 中关于 `size` 的分支，也可以作为较大 chunk 与 top chunk 之间的隔断，避免释放大于 `0x400` 的 chunk 时直接向 top 合并，从而保留 unsorted-bin 元数据用于 libc 泄露。

### 触发 `move` 的别名与 UAF

认证解锁后，`move(idx1, idx2)` 先无条件复制整条记录：

```c
chunk[idx2] = chunk[idx1];
size = malloc_usable_size(chunk[idx1].ptr);
ptr = malloc(size);
ret = snprintf(ptr, size, "%s", chunk[idx1].ptr);

if (ret == 0) {
    puts("move failed!");
    return;
}
```

若源内容第一个字节就是 `\0`，`snprintf` 返回 0，函数在修正目标指针和清空源记录之前退出。可以在 add/edit 交互中发送原始 NUL 字节构造这种内容。返回后 `idx1` 与 `idx2` 仍指向同一旧 chunk，而函数中新申请的副本指针丢失。

删除其中一个别名会释放旧 chunk，另一个槽位仍保存其地址和大小，于是 show 提供 UAF 读、edit 提供 UAF 写。对 tcache chunk，UAF 读可恢复 safe-linking 后的链指针并推导堆基址；对用最小隐藏 chunk 隔开的较大 chunk，释放后可从 unsorted-bin 的 `fd`/`bk` 泄露 `main_arena`，进而计算 libc 基址。

### 从 UAF 放大到控制流

拿到 heap 与 libc 基址后，可以按实际 glibc 版本选择后续路线。作者测试的是 House of Some：利用可控的堆链表写把 `_IO_list_all` 指向伪造 FILE 结构，先让 FILE 操作执行受控 `read`/`write`，泄露 `environ` 指向的栈地址，再把最终 ROP 链写到返回栈。该路线适用于新版 glibc 已移除传统 malloc/free hook 的情况，也绕开了 Full RELRO。

本题到达该阶段所需的题目特有前置条件已经明确：

```text
sort OOB → auth="rf"
realloc(NULL,0) → size=0 的隔断 chunk
move 空字符串失败 → 双指针别名
delete 一个别名 → UAF read/write
unsorted bin → libc leak
tcache / bin 链表操作 → FILE 结构控制
House of Some → 栈泄露与栈上 ROP
```

House of Some 的 FILE 字段、large-bin 写目标、`_IO_list_all`、`environ` 和 gadget 偏移都严格依赖远程 glibc。源仓库没有提供对应 libc 和 exp，无法可靠恢复这些常量；把任意一组公开模板偏移直接填入会生成不可复现的伪脚本，因此这里保留原语与利用阶段，不冒充已验证的最终 payload。

## 方法总结

- 核心技巧：把排序边界错误转化为全局认证字符串覆盖，再利用 move 的错误处理顺序制造堆别名和 UAF，最终通过现代 glibc 的 FILE 利用路线接管控制流。
- 识别信号：结构体数组后紧邻敏感全局变量、排序循环访问 `array[count]`，以及“先发布目标对象、后复制、失败时直接返回”的 move/copy 逻辑，都是高价值内存破坏信号。
- 复用要点：认证覆盖会同步破坏 `last_size`，必须先完成堆布局；`realloc(NULL,0)` 与 `realloc(ptr,0)` 的语义不同；现代 glibc 下所有 tcache 与 FILE 偏移都要和目标 libc 匹配，不能跨版本照抄。
