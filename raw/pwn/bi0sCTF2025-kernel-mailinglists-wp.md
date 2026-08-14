# Kernel-MailingLists

## 题目简述

`Kernel-MailingLists` 是一个 Linux 内核模块题目，服务通过 `/dev/mailbox` 提供 IOCTL 接口（`INITBOX / SENDMAIL / RECVMAIL / SUBSCRIBE / UNSUBSCRIBE`）。内核态维护 mailbox、mail 链和广播订阅链，且大量对象来自 slab 缓存。

官方 `README` 直接给出 intended bug：广播链表解除链接时把 `mail_head->lhead` 当作合法字段处理，导致指针写污染可用于越权写。exploit 中随后进行 kmem 装箱分层泄漏与 cred 篡改拿 root。

由决定性步骤可见这是标准内核提权链，不是配置性多主机横向：既有可控的内核堆/链表 unlink 原语，又有跨进程 uid/gid 提权收口，因此归入 `pwn`（内核利用子类）。

## 解题过程

### 关键观察

官方 exploit/内核源码里可直接对齐三层信息：

1. `mail_utils.c` 的 unlink 逻辑在 `remove` 与 `unlink_user_frame` 里会沿 `lhead` 写回链表头：

```c
if(head->lhead) {
    if(type == REGULAR_MAIL)
        head->lhead->rmailinglist = head->next;
    else
        head->lhead->outmails = head->next;

    head->next->lhead = head->lhead;
}
```

2. `subscribe / unsubscribe` 形成广播列表 `bmailinglist`，`sendmail` 在广播路径会把同一 `mail_head` 复制到多个订阅者，`unlink` 在不同上下文之间触发时会产生未满足假设的指针关系。
3. `admin/exploit/README.md` 的意图描述与 exploit 注释一致：
   “broadcast mailing list sets `mail_head->lhead` even if `lhead` is not initialised… arbitrary write of pointer points to your next mail… cross cache… cred”.

4. 目标内核环境可用 IOCTL 接口通过 `ctypes` 风格的 wrapper 与 `mail_interface.c` 交互，说明 exploit chain 与内核接口是闭环的。

### 利用链

1. **建立可控 unlink primitive 与堆泄漏**

exploit 先批量创建 mailbox 并进行订阅发送，随后对广播/普通邮件组合进行分层释放：

```c
int r1 = 10, r2 = 11;
void do_bcastmail_setup(...){
    for ... submailbox(r1,i);
    sendmail_from_box(i,r1,BROADCAST_MAIL,sz,buf);
    unsubmailbox(r1,i);
    ...
}
```

`leak_mail_slab()` 通过特定顺序 `recvmailfrag(4,ALLMAIL,...)` 后读取堆中 payload 的字段，拿到 allocator slab 指针：

```c
if((*((uint64_t *)mail2->data + 1) != 0))
    slab_leak = *((uint64_t *)mail2->data + 1);
...
printf("[+] Got allocator slab leak %p\n",(void *)slab_leak);
```

2. **校准与 cross-cache**

`calc_slab_base_ptr()` 根据低字节对齐偏移 (`0x00,0x08, ...`) 还原 slab 基址；`cross_cache()` 则用 `__clone`+大量 `pipe()` 将 order-3 物理页拆分/扰动，制造页级再分配可控性。

3. **找目标页并重占用 cred 页**

`try_write()` 在目标地址附近执行 unlink 写，配合 `pipe` 管道内容比对找到“谁被我们的页地址改写”：

```c
try_write(slab_aligned + 0x8,ROTATE,0);  // 向目标页施加 unlink 写
for (int i=0;i<100;i++) {
    read(pipe_desc[i][0],buf,0x20);
    if (((long *)(buf))[1] != 0x6161616161616161) {
        pipe_page = i;
        ...
    }
}
```

定位到目标页后，先关闭若干 pipe，再大量 `setuid(0)` 让同一区域重分配 cred 结构，进入凭据污染阶段。

4. **凭据空字节写入并提权**

`try_write()` 重复调用时把 LSB 旋转写入，找到可落成 `0` 的字节后，循环 `0x28` 次把 `uid/gid/fsuid` 等相关字段置零，最后触发 shell：

```c
for (int i=0;i<0x28;i++) {
    try_write((slab_aligned + 0x8) + i,rotate,state);
}
*shmem = STAGE3_SPAWNSHEL;
setuid(0);
execve("/bin/sh",argc,argv);
```

官方 exploit 尾部也明确写了从 `try_write` 到 `setuid`/跨进程协同的完整主线。

### 验证

可直接用源码里的信息核对三类中间证据：

- 漏洞是否在模块里：`Kernel-MailingLists/admin/src/chall-src/mail_utils.c` 与 `mailinglist.h` 的 `mail_head`/`lhead`/`bmailinglist` 定义；
- exploit 是否按预期打印关键状态：`[+] Got allocator slab leak`、`[+] Found pipe ...`、`[+] Occupied the page with cred objects`；
- 最终权限变化点：`try_write` → `setuid(0)`，`execve("/bin/sh")`。

## 方法总结

- 核心技巧：典型内核 `unlink` 原语。借助广播链路让 `lhead` 关联错位，把一个可控写点扩展为任意页级对象可达，再用 cross-cache + pipe/cred 复用链实现提权。
- 识别信号：看到 IOCTL mailbox/list/双向链表对象、广播订阅关系和 kernel slab 操作时，优先追踪是否出现 `prev/next/lhead` 写回路径。
- 复用要点：`mailbox`/`mail_head` 这种多表关系最容易在 `remove`、`unsubscribe`、`unlink` 分支出现旧指针写回；不要把该类题目归为纯“应用逻辑”。
- 最终分类：`Pwn`（内核利用）。
