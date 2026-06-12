# d3dev && d3dev-revenge

## 题目简述

本题是 QEMU 虚拟设备逃逸题，包含 `d3dev` 和修正版/复仇版。虚拟设备通过 MMIO/PMIO 暴露读写接口，漏洞点在 `d3dev_mmio_write`：写入位置为 `opaque->seek + (addr >> 3)`，而 `seek` 可通过 `d3dev_pmio_write(addr == 8)` 被用户控制。

题目附件包含 QEMU 设备、revenge 附件包和 exp，关键对象是自定义设备的 `blocks` 数组、`seek` 状态和函数指针字段。当 `seek` 被设置到合适值后，`blocks` 数组越界可覆盖结构体后方的 `rand_r` 函数指针；再配合 `r_seed` 和 `key` 的生成逻辑，把调用改成 `system("/bin/sh")` 完成逃逸。

## 解题过程

题目源码和 exp 见 [`yikesoftware/d3ctf-2021-pwn-d3dev`](https://github.com/yikesoftware/d3ctf-2021-pwn-d3dev)。下文保留了设备结构体布局、`seek` 控制越界写和覆盖 `rand_r` 函数指针的关键路径。

漏洞位置很明显，在 `d3dev_mmio_write` 中可以看到，通过 MMIO 向 `opaque->blocks` 写入数据时使用的是：

```c
void d3dev_mmio_write(d3devState *opaque, hwaddr addr, uint64_t val, unsigned int size) {
    pos = opaque->seek + (unsigned int)(addr >> 3);
    if (opaque->mmio_write_part) {
        ...
    } else {
        opaque->blocks[pos] = (unsigned int)val;
    }
}
```

`addr` 和 `val` 都是用户可控的。虽然 `addr` 不能直接超过 MMIO 内存范围来达到溢出，但如果能控制 `seek` 的大小就可以做到。

查看d3dev_pmio_write 发现seek是可以通过令addr==8 直接控制的:

    if ( addr == 8 )
    {
      if ( val <= 0x100 )
        opaque->seek = val;
    }

溢出思路可行之后,观察d3devState 结构体:

```text
d3devState (sizeof=0x1300)
0x000  pdev
0x8e0  mmio
0x9d0  pmio
0xac4  seek
0xad4  r_seed
0xad8  blocks[257]
0x12e0 key[4]
0x12f0 rand_r
```

可以看到 `blocks` 后面有一个函数指针，该指针保存 `rand_r` 函数地址，并会在 `d3dev_pmio_write` 中被调用：

```c
if (addr == 28) {
    opaque->r_seed = val;
    v4 = opaque->key;
    do {
        *v4++ = opaque->rand_r(&opaque->r_seed, 28, val, size);
    } while (v4 != (uint32_t *)&opaque->rand_r);
}
```

在这个分支中rand_r 使用opaque->r_seed 作为参数生成128位的opaque->key ;

所以只要同时控制好opaque->r_seed 和opaque->rand_r 就可以构造出system("/bin/sh\x00") ;

exp 的关键不是固定地址，而是先通过设备读写接口把 `seek` 调到越界位置，再让后续随机数相关调用走到被覆盖的函数指针。

## 方法总结

- 核心技巧：QEMU 设备结构体越界写，利用 MMIO 写入偏移和 PMIO 控制的 `seek` 组合覆盖设备状态中的函数指针。
- 识别信号：虚拟设备结构体中数组后紧邻函数指针，数组写入索引由多个可控字段叠加得到，PMIO 分支会间接调用结构体内函数指针。
- 复用要点：QEMU 逃逸题要先还原设备状态结构和各寄存器语义，再寻找“数组/缓冲区后接函数指针、回调、MemoryRegion”等高价值字段。

