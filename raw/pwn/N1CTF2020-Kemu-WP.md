# N1CTF 2020 Kemu Writeup

## 题目简述

Kemu 修改了 QEMU 的 NVMe/MSI-X 设备，实现一个可通过 PCI BAR 操作的加解密状态结构。`CryptState` 依次包含 `key[0x80]`、`input[0x80]`、`output[0x80]`，随后是函数指针和状态字段。设备代码对固定长度缓冲区调用 `strlen()`，只要写满非零字节，长度计算就会越过数组边界。该越界既能泄露 QEMU 地址，也能覆盖加密函数指针。

## 解题过程

### 映射设备 BAR

来宾系统把 PCI 设备的 BAR 暴露为 sysfs resource 文件。打开对应设备的 `resource4` 并 `mmap()` 后，即可按题目定义的寄存器偏移读写设备状态：

```c
int fd = open("/sys/bus/pci/devices/0000:00:04.0/resource4", O_RDWR | O_SYNC);
void *bar = mmap(NULL, BAR_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
```

设备号和 BAR 编号应通过 `/sys/bus/pci/devices` 实际枚举确认。

### 越界读取泄露 QEMU 基址

向 `input` 写入恰好 `0x80` 个非零字节，不放终止 NUL。设备对输入或输出执行 `strlen()` 时会继续扫描后面的 `CryptState` 字段。再从输出窗口读取超过 `0x80` 的数据，便能取得紧随数组之后的 `crypt_func` 指针。

```text
qemu_base = leaked_crypt_func - known_function_offset
system_plt = qemu_base + system_plt_offset
```

这些偏移来自题目附带 QEMU 二进制。先确认泄露指针确实落在 QEMU 可执行映射内，再计算基址。

### 借加密过程覆盖函数指针

加密逻辑按字节异或 key、input 和 output。由于错误长度大于 `0x80`，循环会继续写到 `crypt_func`。攻击者已知原数据和异或规则，可以反向计算输入，使越界写后的 8 个字节恰好等于 `system@plt`：

```text
cipher_byte = input_byte XOR key_byte
input_byte  = desired_byte XOR key_byte
```

把 key 缓冲区写成：

```text
cat /flag
```

再触发使用 `crypt_func` 的解密路径。原函数签名的第一个参数正好指向 key，覆盖后调用等价于：

```c
system(state->key);
```

命令输出经 QEMU 进程的标准输出返回。

## 方法总结

设备逃逸题应先画出宿主侧状态结构和来宾可访问窗口，再追踪每次 MMIO 操作会读写哪些字段。`strlen()` 用于定长二进制缓冲区会同时造成越界读和越界写；当相邻字段包含函数指针时，两类原语可以自然串成“泄露宿主基址 → 精确覆盖控制流”。
