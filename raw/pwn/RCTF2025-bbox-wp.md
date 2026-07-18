# bbox

## 题目简述

题目在 QEMU 中挂载自定义 PCI 设备 `virtsec-device`。访客机可把 `0000:00:04.0/resource0` 映射为 0x2000 字节 MMIO：低 0x1000 是命令寄存器，高 0x1000 是 block 数据窗口。设备允许分配、读取和合并 16 字节 block，并提供一个 gift 寄存器用于调用宿主侧函数。

核心漏洞是 block 的物理位置与逻辑 size 脱节。每个 block 的 `data_offset` 固定为索引乘 0x10；merge 只把目标 block 的 size 累加，却没有检查合并后是否越过 0x100 字节 `data_buffer`。因此同一缺陷同时提供宿主堆对象的越界读写，最终可覆盖紧邻缓冲区的函数指针与参数，实现 QEMU 进程代码执行。

## 解题过程

### 1. 还原 MMIO 协议和宿主对象布局

访客侧需要的主要寄存器如下：

| 偏移 | 含义 |
| --- | --- |
| `0x0c` | 写入命令号 |
| `0x14` | 当前 block id |
| `0x18` | 新 block size，正常范围 `1..0x10` |
| `0x24` | block 数量 |
| `0x30`、`0x34` | 待合并的两个 block id |
| `0x38` | 触发 gift 调用 |
| `0x1000..0x1fff` | 当前 block 的数据窗口 |

命令 2 分配 block，命令 3 准备读取，命令 4 合并，命令 6 复位。MMIO 以 32 位小端访问；访客侧封装只需先写 block id/size，再写命令寄存器。

单题 WP 给出的 `VirtSecDevice` 关键尾部布局为：

```c
uint8_t data_buffer[16 * 16];              // 0x100 bytes
int (*gift_function)(const char *fmt, ...);
const char *gift_param1;
uint8_t encrypted_buffer[16 * 16];
```

设备初始化后，`gift_function` 指向宿主 libc 的 `printf`，写 gift 寄存器会执行近似 `gift_function(gift_param1)` 的调用。于是 `data_buffer + 0x100` 正是最有价值的越界目标。

### 2. 反复 merge 扩大逻辑 size

总 PDF 的简洁构造是先分配 block 0、1，各 0x10 字节并执行 `merge(0, 1)`，令 block 0 的 size 变成 0x20；之后反复重新分配 block 1 并合并到 block 0：

```c
alloc(0, 0x10);
alloc(1, 0x10);
merge(0, 1);
for (int i = 0; i < 0x11 - 2; i++) {
    alloc(1, 0x10);
    merge(0, 1);
}
```

最终 block 0 的逻辑 size 达到 0x110，但起点仍是 `data_buffer + 0x00`。另一种等价构造是先填满 16 个 block、把它们合并成 0x100 字节大块，再合并一个额外 block 扩到 0x110。决定性条件都是“size 增长，固定 offset 不变”。

### 3. 越界读泄露宿主 libc

先触发一次 gift，确保函数指针及参数处于已初始化状态，再从 block 0 的数据窗口读取 0x110 字节。偏移 `0x100` 的 qword 即 `gift_function`，在附件环境中是 `printf`：

```c
uint64_t printf_addr = *(uint64_t *)(buf + 0x100);
uint64_t libc_base   = printf_addr - 0x606f0;
uint64_t system_addr = libc_base + 0x50d70;
uint64_t binsh_addr  = libc_base + 0x1d8678;
```

这些偏移来自题目附带的 Ubuntu 22.04 宿主 libc，只能用于该环境。复现时应先确认越界 qword 确实落在 libc 映射，并用对应 libc 符号重新计算。

### 4. 越界写覆盖 gift 调用

接下来要让一次 merge 的复制目标越过 `data_buffer` 尾部。稳定做法是：

1. 为多个后续 block 写入重复的 16 字节载荷 `[system_addr, binsh_addr]`。
2. 从靠近缓冲区尾部的固定 offset 开始反复 merge，使逻辑写位置依次推进到 `0x100` 和 `0x108`。
3. 通过越界读回这 16 字节，确认 `gift_function == system` 且 `gift_param1 == binsh_addr`；若状态不符，复位会话并重新建立全部 block，而不是盲目多跑一次。
4. 再写 gift 寄存器，宿主执行 `system("/bin/sh")`，得到运行 QEMU 的宿主用户 shell，读取容器根目录的 `/flag`。

总 PDF 的 exploit 将 `[system, "/bin/sh" 指针]` 写入 block 1 至 15，再合并 block 2 与后续 block，把载荷推过边界。PDF 注释称偶尔需运行两次或多写一次；原因是它没有在触发前验证设备内部 block 状态。单题 WP 的更稳版本会先填满连续 block、检查每次命令的 error code，并在调用 gift 前回读覆盖结果。

仓库给出的 flag 为：

```text
RCTF{escape_4a7f9b2e1c8d3a5f6b0e9c2d1a8f37}
```

## 方法总结

- 这是一条 QEMU 设备逃逸链：访客只操作 PCI BAR，越界读写实际发生在宿主 QEMU 进程的 `VirtSecDevice` 对象中。
- 根因不是单次 MMIO 长度过大，而是 merge 后的逻辑 size 未与固定物理 offset 和 0x100 字节总缓冲区重新校验。
- 利用顺序应固定为“增长 block → 越界读 `printf` → 计算宿主 libc → 越界写函数指针与参数 → 回读验证 → 触发 gift”。
- 所有 libc 偏移、设备路径和结构体相对位置均绑定附件；若回读不符，应重建 block 状态并检查 error code，不应把偶然成功当成稳定原语。
