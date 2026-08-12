# Watchdog

## 题目简述

题目在 FPGA 中运行一个 RV32I 软核，通过 UART 接收自定义 `.wdog` payload 并从 `0x00008000` 执行。flag 位于只读 MMIO 区，但 watchdog 会限制程序运行时间。需要先找到并关闭或喂停 watchdog，再从 flag MMIO 读取数据并经 UART 输出。

## 解题过程

`env.zip` 给出最小交叉编译环境和关键地址：

```c
#define UART_BASE  0x10000000u
#define LED_BASE   0x10000100u
#define FLAG_BASE  0x20000000u
```

官方解法对紧邻已知外设的 `0x10000200` 至 `0x100002fc` MMIO 窗口按 4 字节步长写零。该有界 fuzz 会命中 watchdog 控制寄存器并使其停止，而不会盲扫整个地址空间：

```c
for (unsigned int addr = 0x10000200u;
     addr < 0x10000300u;
     addr += 4u) {
    *(volatile unsigned int *)addr = 0;
}
```

flag 数据从 `FLAG_BASE` 开始，字节长度保存在 `FLAG_BASE + 0xf0`。先读长度并向上取整为 32 位字数，再以小端顺序拆开每个 MMIO word：

```c
unsigned int length = read32(FLAG_BASE + 0xf0u);
unsigned int words = (length + 3u) / 4u;

for (unsigned int i = 0; i < words; i++) {
    unsigned int value = read32(FLAG_BASE + 4u * i);
    uart_putc(value & 0xffu);
    uart_putc((value >> 8) & 0xffu);
    uart_putc((value >> 16) & 0xffu);
    uart_putc((value >> 24) & 0xffu);
}
uart_putc('\n');
```

UART 发送前轮询 `UART_BASE + 0x04` 的 bit 0，准备好后把字符写到 `UART_BASE + 0x00`。payload 使用 `-march=rv32i -mabi=ilp32 -nostdlib -ffreestanding` 编译，链接地址为 `0x8000`，最大 16 KiB。

生成的裸二进制还要封装为：

| 偏移 | 内容 |
| ---: | --- |
| `0x00` | ASCII `WDOG` |
| `0x04` | 4 字节小端 payload 长度 |
| `0x08` | RV32I 原始 payload |

```python
packet = b"WDOG" + len(payload).to_bytes(4, "little") + payload
```

把 `payload.wdog` 放到 `/challs/watchdog/payload.wdog`，运行题目提供的上传脚本。脚本刷入 bitstream，经 UART 逐字节发送 payload，看到 `LOADED` 和 `RUNNING` 后继续打印软核输出。最终得到：

```text
grey{mmio_fuzzz}
```

## 方法总结

这题的核心是先建立软核的内存映射，再把 watchdog 视为可控 MMIO 外设。官方基线没有精确逆出单个控制寄存器，而是在一个已限定的 256 字节窗口中写零，换取简单可靠的停表效果；随后按长度字段读取 flag 并通过轮询 UART 输出。`.wdog` 只是带 magic 和长度的上传封装，不能把 ELF 文件直接交给加载器。
