# bi0sCTF 2024 - virtio-note

## 题目简述

题目提供 Linux 客体中的 `/dev/virtionote` 前端驱动和一个自定义 QEMU virtio 设备。客体通过 ioctl 提交 `idx` 和 0x40 字节缓冲区；QEMU 后端把 `idx` 用作 note 指针数组下标，却只排除了负数而没有检查正数上界。任意大正索引因此可以读写 `VirtIONote` 对象周围的 QEMU 堆。

利用先从越界槽泄漏设备对象地址，再让某个槽指向 note 指针数组本身，构造 QEMU 进程内任意地址读写。随后泄漏 virtqueue 与 QEMU PIE，写入 ORW ROP 链，并覆盖 virtqueue 回调为栈迁移 gadget，从宿主进程中读取 `flag.txt`。

## 解题过程

### 封装设备读写并取得堆泄漏

内核驱动把请求包装成 `{op, idx, addr}` 送到 virtqueue；`VN_READ` 把后端返回的 0x40 字节复制给用户，`VN_WRITE` 则先复制用户数据。用户态可封装：

```c
unsigned long vn_read(unsigned int idx);
void vn_write(unsigned int idx, unsigned long value);
```

由于 QEMU 后端直接访问 `notes[idx]`，读取官方分析确定的正 OOB 索引会命中设备对象附近的堆指针。独立题目的官方 exploit 用 `vn_read(0x3e)` 得到 `VirtIONote` 地址；不同构建可能泄漏对象内指针再减固定偏移，必须以调试版 QEMU 堆布局校准。

### 构造任意地址读写

设备对象中 note 指针数组位于已知偏移。先把一个可越界控制的槽写成该数组地址：

```c
vn_write(0x13, vnote + 0x210);
```

这样，对另一个特殊索引的写入实际上会修改 `notes[0]`。官方原语为：

```c
void arb_write(unsigned long addr, unsigned long value) {
    vn_write(0x1e, addr);  /* notes[0] = addr */
    vn_write(0x00, value); /* *addr = value */
}

unsigned long arb_read(unsigned long addr) {
    vn_write(0x1e, addr);
    return vn_read(0x00);
}
```

每次把目标地址写入指针槽，再通过索引 0 解引用，即得到 QEMU 堆上的 64 位任意读写。

### 泄漏 PIE 并布置 ROP

从 `vnote + 0x208` 读取 `VirtQueue *`。再读取 `vnote + 0xf8` 的 QEMU 代码指针，减去题目二进制中该符号偏移 `0x688840`，得到 PIE 基址。所有 gadget 与 `open/read/write/exit` 地址都由该基址加相对偏移计算。

在 `vnote` 后方可写区域放置：

- `flag.txt\0`；
- 主 ROP：`open` 文件、把返回 fd 交给 `read`、再向标准输出 `write`；
- 一个小型二次栈迁移链。

由于 `open` 的返回值在 `eax`，官方链使用 `xchg edi,eax` gadget 把它变成下一次 `read` 的第一个参数。

### 劫持 virtqueue 回调

把 `vnote->vnq->handle_output` 所在的 `virtq + 0x58` 改为

```text
lea rsp, [rbp - 0x10]; pop r12; pop r13; pop rbp; ret
```

一类 pivot gadget，并在 `vnote->parent_obj` 起始处布置 `pop rsp; ret` 与主链地址。最后发起一次读请求，例如使用异常索引 `0x1337`，QEMU 处理 virtqueue 时进入被覆盖的 handler，完成两次栈迁移并执行 ORW 链。输出来自 QEMU 宿主进程的当前目录，而不是客体文件系统。

## 方法总结

正数 OOB 发生在 QEMU 设备后端，因此其影响边界是宿主 QEMU 进程。一次邻接堆泄漏定位设备对象，把 OOB 指针槽回指到 note 数组便可自举为任意读写；随后通过对象内 virtqueue 和代码指针解决堆与 PIE 地址。最终覆盖回调并执行 ORW ROP，比在 QEMU sandbox 下尝试启动新进程更可靠。
